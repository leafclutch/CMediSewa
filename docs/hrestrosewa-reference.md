# HRestroSewa: Architecture Analysis & Reuse Matrix

Analysed 2026-11-20 against HRestroSewa `main` @ `a12d2d5` (sibling folder `../RestroSewa`).
Everything under **Findings** was verified by reading the code, not only file names or docs.
File references point into the HRestroSewa repo. **Nothing in HRestroSewa was modified.**

## 1. Findings: how HRestroSewa actually works

### Stack (confirmed: `package.json`)
Next.js App Router (16.x per its memory bank) · React 19 · TypeScript · Tailwind v4 · `radix-ui` ·
`lucide-react` · `@supabase/ssr` + `supabase-js` · `web-push` · `nodemailer` · `pdf-lib` ·
`qrcode.react` · `pg`. Most dependencies are pinned to `latest`, so a fresh install can pull in
breaking majors (a risk CMediSewa avoids, see D-16).

### Surfaces (confirmed: `app/`)
Route groups in **one** Next app: `(marketing)` landing (SEO, `robots.ts`, `sitemap.ts`),
`(auth)/login`, `(superadmin)/superadmin/*`, `(admin)/admin/*` (owner + permitted staff),
`(employee)/employee/*` (staff floor dashboard), `(customer)/c/[slug]` (public QR ordering, no
auth). `app/actions/*` (≈30 Server Action modules) **is the API**. `app/api/*` holds only
`realtime` (SSE), `push/action`, `cron/daily-summary`, `cron/subscription-expiry`, `floor`, `_perf`.

### Multi-tenancy (confirmed)
- Shared database, shared schema. `restaurants` is the tenant and every tenant row carries
  `restaurant_id`.
- Isolation is enforced **in application code**. Every Server Action resolves the caller's
  `restaurant_id` from their staff row and scopes each query with it.
- `lib/cache/tenant-cache.ts` is the only cross-request cache. Its tenant id is mandatory in
  both the key and the tag, so it can't be left out by mistake.
- Tenancy is path-based on one domain (`/c/[slug]` for the public side). No subdomains.

### Data access & RLS (confirmed: `lib/realtime/bus.ts`, migration `20260801000001`)
- **All** DB access is server-side through the **service_role** client, which bypasses RLS.
  The browser never queries tables. The migration notes record a check that no client
  component calls `.from(`.
- RLS is **enabled on every table**, and `anon`/`authenticated` have **all grants revoked**
  (including default privileges for future tables). The initial schema's "staff read own"
  policies still exist but cannot be reached. RLS is a backstop, not the authorization layer.
- Integrity- and money-critical operations are **Postgres RPCs** (one transaction each),
  granted to `service_role` only. They raise bare error codes that actions map to friendly text.
- Lesson recorded there: grants and the anon lockdown belong in migrations. The self-hosted
  Supabase image grants `anon`/`authenticated` full CRUD by default.

### Authentication (confirmed: `proxy.ts`, `lib/auth/*`, `app/actions/auth.ts`)
- Supabase Auth sessions in cookies via `@supabase/ssr`.
- `proxy.ts` (Next middleware) is a redirect optimisation only. It checks whether a valid
  session exists and never makes authorization decisions. Its matcher excludes static
  assets, `/api` and `/c/`.
- `getAuthUser()` verifies the JWT **locally** with `getClaims()` (cached JWKS). On a
  verification error it falls back to `getUser()`; it never treats an error as "signed out".
  It is memoised per request with React `cache()`.
- `getStaffRow()` reads `restaurant_users` **live on every request** (`is_active`,
  `deleted_at`), so deactivating someone takes effect immediately. It throws on a query error
  rather than returning null, after a past redirect-loop incident.
- Owner/admin: email + password. Staff: **4-digit PIN** stored as the password of a synthetic
  Auth account `emp-<id>@restrosewa.internal` (`pinToAuthPassword` prefix mapping), protected
  by a DB lockout (`pin_lockout_status`: 5 failures → 30 min).
- Super admins are a separate `super_admins` table (`isSuperAdmin`), not a tenant user.
- Logout is local scope (this device only), because of an iOS PWA storage-container issue.

### Roles & permissions (confirmed: `lib/permissions.ts`, `lib/auth/guards.ts`)
- `role` is either `restaurant_admin`, which **bypasses every permission**, or
  `restaurant_employee`, which holds `permissions text[]`.
- `PERMISSIONS` is the single source of truth for permission keys. `ACCESS_SECTIONS` groups
  them into per-module **levels** (None/View/Full…) plus **extras**, so the staff editor shows
  one control per module. Write implies read. Presets exist for common roles.
- Guards: `requireSuperAdmin`, `requireRestaurantStaff`, `requireAdminOrPermission(p)`,
  `requireRestaurantAdmin`. **Every page and every action re-checks on the server.** Navigation
  is generated from permissions, so it always matches what the guards allow.
- Record visibility: "assignment" (`lib/assignments.ts`). Staff see only the table groups and
  rooms assigned to them. Only the admin sees everything. This is a working example of
  record-level scoping layered on top of capability permissions, and it is the direct model
  for CMediSewa's care-relationship rule.
- Sensitive edits sit behind an admin **Security PIN** and are written to `security_audit_log`
  (actor, operation, target, outcome, before→after JSON).
- Module availability also follows the tenant's business type (`restaurants.type` →
  `hasRooms`). A module the business doesn't have can't be navigated to, granted or posted to.

### Realtime (confirmed: `lib/realtime/bus.ts`, `app/api/realtime/route.ts`)
DB triggers `NOTIFY rs_events` with a topic name only. Each process holds one Postgres
`LISTEN` connection on a *session* connection, because the transaction pooler drops
cross-connection NOTIFY. The bus fans events out per tenant over SSE with a 25 s heartbeat.
Clients receive `{topic}` only and refetch through permission-checked actions. Supabase
Realtime was deliberately rejected because it would require opening RLS to the browser.

### Push notifications (confirmed: `lib/push/send.ts`)
web-push with VAPID. Recipients are resolved through **the same visibility predicate** as the
in-app panel. Sending happens *after* the response (`after()`), so the user who triggered it
isn't kept waiting. In-app panel = record; push = signal.

### Email & scheduled tasks (confirmed: `lib/email/mailer.ts`, `supabase/cron/*`, `app/api/cron/*`)
- There is one platform sender, and tenants only choose recipients. **Resend over HTTPS** is
  used when `RESEND_API_KEY` is set, otherwise Gmail SMTP. The VPS host silently drops
  outbound SMTP on ports 25/465/587. The mailer never throws, and retries with backoff.
- `pg_cron` + `pg_net` POST to secret-gated routes (`x-cron-secret`, constant-time compare,
  404 on failure). The URL and secret come from Supabase **Vault**. Cron SQL is applied by hand
  per environment and kept out of `migrations/`.
- `report_deliveries` guarantees each daily report is sent exactly once. The PDF is built with
  `pdf-lib`, because pdfkit doesn't bundle in Next.
- Subscription expiry is computed from `install_date` + `subscription_extra_days` + a 365-day
  cycle (`lib/subscription.ts`) and stored nowhere. A daily cron deactivates expired tenants.

### Storage (confirmed: `app/actions/branding.ts`, `menu-images.ts`)
Public buckets (`restaurant-logos`, `menu-images`) receive uploads through the service client
with uuid filenames that are never overwritten, and are cached for 31 days by `next/image`.
Server Action body limit is 3 MB. No private-file pattern exists yet.

### PWA (confirmed: `public/sw.js`, `components/pwa/*`)
Manifest + service worker that caches **nothing** dynamic (only the offline page and static
icons/splash). Authenticated responses are never intercepted ("served to another is a data
leak"). `OfflineGate` refuses writes while offline instead of queueing them.
`sw.js` is served `no-store` so a new deploy always reaches users.

### Deployment (confirmed: memory bank decisions 2026-08-01 → 2026-10-04, runbook)
- App and **self-hosted Supabase** both run under **Coolify** on an OVHcloud VPS.
- Postgres has no public port. Tooling reaches it through Kong's `/pg/query` (postgres-meta)
  with the service-role key, via `scripts/lib/pg-http.mjs`.
- `scripts/migrate.mjs` applies ordered SQL migrations with `status`/`up` commands and three
  production interlocks (`--prod`, `--yes`, hostname check). The rule is DB before app, and
  every migration is additive.
- Nightly `pg_dump` from a systemd timer keeps 14 copies, each checked with `pg_restore --list`.
  Off-site copies are **not confirmed** in the docs.
- Dev runs on a hosted Supabase project, prod on self-hosted.

### Conventions worth keeping
Derived values are computed, never stored (stock, balances, subscription days). Date
arithmetic for the Nepal timezone lives in exactly one module (`lib/business-day.ts`).
Pure helpers stay out of `"use server"` modules (exporting them causes a runtime 500).
Performance is bounded by round-trips, not query cost: remove round trips, batch,
`Promise.all`. The memory bank's decisions log is append-only.

## 2. Reuse matrix

### A. Reusable patterns (adapt the idea, re-implement for CMediSewa)
| Pattern | HRestroSewa source | CMediSewa adaptation |
|---|---|---|
| Shared-schema multi-tenancy | `restaurant_id` everywhere | `clinic_id` everywhere, **plus composite FKs `(clinic_id, id)`** so the DB itself rejects cross-tenant references |
| Server-only data access + RLS deny-all backstop | service_role, revoked anon grants | Same, **plus** clinical access through RPCs that check care relationship and write the audit row in one transaction |
| Middleware = redirect only; guards = authority | `proxy.ts`, `guards.ts` | Same split |
| Live staff-row check per request | `getStaffRow` | Same, plus a check that the clinic is active and its subscription valid |
| Permission sections/levels/presets + derived nav | `lib/permissions.ts` | New medical permission keys and presets (doctor, nurse, receptionist…) |
| Record-level visibility predicate | `lib/assignments.ts` | **Care relationship** (`lib/care-access.ts`), shared by pages, push and exports |
| Business-type module gating | `hasRooms` | Plan/feature gating (lab, pharmacy) by plan entitlement |
| Security PIN + `security_audit_log` | security-pin module | Generalised `audit_log` for *reads* of clinical data as well as writes |
| SSE over LISTEN/NOTIFY, topic-only | `bus.ts` | Queue board, appointment list, check-in signals |
| Push through the same visibility predicate, after response | `push/send.ts` | Same, and payloads **never contain PHI** |
| Exactly-once scheduled sends | `report_deliveries` | `notification_deliveries` for reminders |
| pg_cron → secret-gated route | `supabase/cron/*` | Reminder dispatch, subscription expiry, retention jobs |
| Derived, not stored | stock, finance, subscription | Invoice balance, slot availability, subscription status |
| Sequential trigger-stamped bill numbers | `payments.bill_number` | Invoice numbers **per clinic per Nepali fiscal year** |
| Migrations own grants/RLS from day one | `20260801000000/1` | Migration 0001 sets the access posture before any table exists |
| Ordered SQL + guarded runner | `scripts/migrate.mjs` | Same tool shape, with CMediSewa hostnames |

### B. Reusable components (copy then adapt; no shared package)
Recommendation: **copy at Phase 0, then own the copy.** A shared npm package or monorepo
would tie CMediSewa's release cadence to a production POS for a handful of small files.
Revisit if more than about 3 products share them (D-02).

- `proxy.ts` (cookie-preserving redirect, matcher discipline)
- `lib/auth/current-user.ts`, `lib/auth/guards.ts`. Same shape, new tables.
- `lib/supabase/{server,service,admin}.ts`
- `lib/cache/tenant-cache.ts`
- `lib/realtime/bus.ts`, `lib/realtime/use-realtime.ts`, `app/api/realtime/route.ts`.
  The customer-session branch is removed.
- `lib/push/*`, `lib/pwa/*`, `components/pwa/*` (offline gate, install prompt, SW registration),
  `public/sw.js` posture
- `lib/email/mailer.ts` + `resend-payload.ts`
- `lib/reports/pdf/*` document chrome (pdf-lib) for prescriptions, receipts and reports
- `lib/business-day.ts` (Nepal UTC+05:45 handling), `lib/currency*.ts`, `lib/subscription.ts`
- `components/ui/*` primitives (button, input, skeleton, search-select, period-filter,
  theme toggle/sync, count-pill, payment-method-picker)
- `scripts/migrate.mjs`, `scripts/lib/pg-http.mjs`, `scripts/generate-pwa-assets.mjs`
- PIN lockout SQL (`pin_lockout_status`), used for password attempts

### C. Must be designed fresh for healthcare
Patients and MRNs · doctors with professional registration (NMC number) · departments and
services · doctor schedules, leave and slot generation · appointments with a status machine,
double-booking prevention and an online booking abuse model · token/queue check-in ·
encounters (vitals, complaint, diagnosis, notes) with finalize/addendum immutability ·
prescriptions · allergies and medical history · lab orders and results · clinical file
storage (private, signed URLs) · **care-relationship access** and break-glass ·
**read-access audit** · consent, retention and patient data rights · clinical invoices (health
services, likely VAT-exempt but unverified) · PHI-minimal notification content · patient
self-service links.

### Explicitly NOT carried over
Tables, sessions, orders, OT tickets, workstations, menu, stock/recipes, rooms/stays, walk-ins,
mock bills, rota payroll, QR ordering modes, the 4-digit-PIN-as-primary-credential model
(too weak for medical records, see D-05), and the admin-bypasses-everything rule for clinical
content (see D-06).

## 3. Observed HRestroSewa risks that CMediSewa should not inherit
These are observations, not changes requested to HRestroSewa.
1. `latest` dependency ranges mean builds aren't reproducible.
2. Kong `/pg/query` is publicly routable with superuser reach behind the service-role key.
   CMediSewa should restrict it by IP allowlist or disable it, and use an SSH tunnel for tooling.
3. Off-site backups aren't documented.
4. Schema drift happened between the live DB and the migrations. CMediSewa should verify
   parity in CI against a fresh DB replay.
