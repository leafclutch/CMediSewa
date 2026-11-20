# Architecture

Status: **Recommendation** (proposed design, not yet implemented). Reasons are in
`decisions.md`. The HRestroSewa patterns referenced here are verified in
`../docs/hrestrosewa-reference.md`.

## 1. System overview
```
 Patients (phone browser)       Clinic staff (PWA: phone/tablet/PC)       Super Admin
          │                                   │                                │
          ▼                                   ▼                                ▼
 ┌─────────────────────────── Next.js app (one deployment, Node runtime) ───────────────────┐
 │ proxy.ts: session refresh + redirect only (never authorizes)                              │
 │ Route groups: (marketing) (public)/c/[slug] (auth) (clinic)/clinic (superadmin)           │
 │ Server Components (reads) + Server Actions in app/actions/* (the API)                    │
 │ lib/auth (guards) · lib/permissions · lib/care-access · lib/audit · lib/scheduling        │
 │ app/api: realtime (SSE) · cron/* (secret-gated) · push/action · files/[id] (signed URL)   │
 └───────────────┬───────────────────────────────┬──────────────────────────────┬───────────┘
                 │ service_role (server-only)    │ LISTEN cm_events             │ HTTPS
                 ▼                               ▼                              ▼
 ┌──────── Self-hosted Supabase (Coolify) ────────┐                   Resend · SMS (P-05)
 │ Postgres: tables (RLS on, no anon/auth grants) │                   web-push services
 │   RPCs for integrity + clinical access + audit │
 │   pg_cron + pg_net → /api/cron/*  (Vault)      │
 │ Auth: email+password, TOTP MFA                 │
 │ Storage: public (logos, doctor photos)         │
 │          private (clinical files, signed URLs) │
 └────────────────────────────────────────────────┘
```

## 2. Frontend architecture
- **One Next.js App Router application** (D-01). Route groups each have their own layout:
  - `(marketing)`: static/SEO landing pages and pricing.
  - `(public)/c/[slug]`: the clinic's patient-facing website. It has its own layout and
    theme, imports nothing from `(clinic)`, and calls only `app/actions/public/*` (§6).
  - `(auth)`: staff login, MFA challenge, password reset.
  - `(clinic)/clinic/*`: the single dashboard for Clinic Admin and staff (D-07). Navigation is
    derived from permissions (`getClinicNav(ctx)`) the same way `getStaffNav` works in
    HRestroSewa.
  - `(superadmin)/superadmin/*`: the platform operator.
- **Server Components fetch data and Client Components handle interaction.** Clients call
  Server Actions again and listen to `useRealtime` for refresh signals.
- Admin and staff share the same client components. A screen is never duplicated per role;
  only what it shows changes.
- UI kit: Tailwind v4 tokens + radix primitives, copied from HRestroSewa's `components/ui`
  and restyled (`ui-ux.md`).
- **i18n-ready from day one** (P-13). All user-visible strings go through a message lookup,
  so Nepali can be added without touching components. English ships first.
- **Dates:** timestamps are stored in UTC and shown in the clinic's timezone (Asia/Kathmandu,
  +05:45), with BS (Bikram Sambat) shown next to AD where patients see dates. All date
  arithmetic lives in one module, `lib/clinic-time.ts`, following HRestroSewa's
  `business-day.ts` rule (D-14).

## 3. Backend architecture
- **Server Actions (`app/actions/*`) are the API.** Every action:
  1. resolves the caller with `getClinicUser()` (memoised per request),
  2. checks the capability permission (`hasPermission`),
  3. for patient data, checks the record-level care relationship (`lib/care-access.ts`) or
     calls a clinical RPC that does so in SQL,
  4. runs tenant-scoped queries through the service client, always filtered by
     `ctx.clinic_id`,
  5. writes an audit entry for sensitive reads and writes (`lib/audit.ts` or inside the RPC).
- **Postgres RPCs** handle anything that must be atomic: booking an appointment (conflict
  check + insert), check-in (token assignment), finalizing an encounter, issuing an invoice
  (number stamping), recording payment, and reading or writing clinical records along with
  their audit row. They are granted to `service_role` only and return bare error codes
  (`SLOT_TAKEN`, `ENCOUNTER_FINALIZED`), which actions turn into friendly messages.
- **A tenant-scoped data helper** (`db(ctx)`) wraps the service client so that clinic-scoped
  tables can't be queried without `clinic_id` in the filter. This is a code-review aid; the
  database-level guarantees are composite FKs and RPC checks (D-03).
- Pure helpers live in plain modules, never exported from `"use server"` files.
- **Derived, not stored:** slot availability, invoice balance, subscription status and queue
  position are computed. The things that are stored are those that must not change after
  the fact: invoice numbers, finalized records, prices snapshotted onto invoice lines.

## 4. Multi-tenant architecture
- **Shared database, shared schema, `clinic_id` on every tenant-owned row** (D-03).
- **Composite foreign keys** `(clinic_id, x_id) → x(clinic_id, id)` on every cross-table
  reference. An appointment of clinic A therefore cannot point at a patient or doctor of
  clinic B, even if application code has a bug. This is the main structural upgrade over
  HRestroSewa.
- **Identity vs membership:** an Auth user (`auth.users`) links to clinics through
  `clinic_members` (one row per user per clinic). The MVP UI assumes one clinic per login,
  but the model allows a visiting doctor to belong to several clinics later (P-16). The
  active clinic is part of the request context.
- **Tenant lifecycle:** `clinics.status` can be `active`, `suspended` or `closed`. The guard
  refuses dashboard access unless the clinic is active and its subscription is valid.
  Public booking stops when the clinic is suspended.
- Caching follows HRestroSewa's `tenantCache`. Only slow-changing, non-PHI configuration is
  cached (clinic profile, departments, services, schedules). **Patient data is never cached
  across requests.**

## 5. Access structure
| Layer | Who | Authority | Sees patient data? |
|---|---|---|---|
| Super Admin | `platform_admins` row | `requireSuperAdmin()` | **No** (D-08) |
| Clinic Admin | `clinic_members.role = 'clinic_admin'` | All operational permissions | Clinical content only through a clinical role or audited break-glass (D-06, P-07) |
| Staff | `clinic_members.role = 'clinic_staff'` | `permissions[]` + care relationship | Only within the care relationship and permissions |
| Patient (public) | anonymous, or holds a booking token | `public/*` actions only | Only their own booking, through the booking token |

Details are in `permissions.md`.

## 6. Patient website architecture
- Lives at `/c/[slug]` on the platform domain in the MVP. Subdomains (`slug.<domain>`) come in
  Phase 2 through a `proxy.ts` host rewrite to the same route, and custom domains later (P-01,
  D-13).
- Public, cacheable pages: clinic profile, departments, services, doctor profiles. They use
  ISR/`revalidateTag` on tenant-tagged caches.
- Dynamic pieces: slot availability (fresh query) and the booking form.
- `app/actions/public/*` is the **only** server code the public site may call. It
  returns allow-listed fields and never returns MRNs or clinical data. The data contract is
  explicit types such as `PublicDoctor` and `PublicSlot`.
- **Booking self-service:** after booking, the patient gets a reference number and an
  unguessable **manage link** (`/c/[slug]/booking/[token]`). The token is 128-bit random and
  stored hashed. It lets the patient view, cancel or reschedule that one appointment and
  nothing else. There is no patient login in the MVP (patient portal is Phase 2).
- Abuse controls: rate limits per IP and per phone, a CAPTCHA (e.g. Cloudflare Turnstile,
  to be confirmed), limits on future bookings per phone, and optional OTP (P-04). See
  `security.md` §8.

## 7. Authentication & authorization
- **Supabase Auth.** Staff sign in with email (or a clinic-issued username mapped to a
  synthetic email) plus a password. No 4-digit PIN as the primary credential (D-05). TOTP MFA
  is mandatory for Super Admins and recommended or mandatory for Clinic Admins and doctors
  (P-11).
- Lockout after repeated failures, reusing HRestroSewa's DB-clock lockout pattern.
- `proxy.ts` refreshes sessions and redirects signed-out users only. Guards (`requireClinicUser`,
  `requirePermission`, `requireClinicAdmin`, `requireSuperAdmin`) re-read the membership row
  **live** on every request: membership active, clinic active, subscription valid. Local JWT
  verification uses `getClaims()` and falls back to the network on error rather than
  returning null.
- **Session policy** (**Recommendation**): idle timeout for clinical screens, re-auth before
  break-glass, and local-scope logout. Shared reception PCs may get a "switch user" quick
  re-auth in a later phase (P-06).

## 8. Realtime
Copied from HRestroSewa: DB triggers `NOTIFY cm_events` with `{clinic_id, topic}`, one
`LISTEN` session connection per process, SSE to clients of that clinic, and **topic names
only**. Clients then refetch through permission-checked actions. Topics: `appointments`,
`queue`, `encounters`, `billing`, `schedule`, `settings`, `staff`, `notifications`. The public
site gets no SSE stream in the MVP and polls its booking status instead.

## 9. Notification architecture
Three channels share one dispatcher (`lib/notify/*`):
| Channel | Audience | Use | Content rule |
|---|---|---|---|
| In-app panel | staff | durable record (`notifications` table) | may reference the patient, since the viewer is already authorized |
| Web push | staff | signal ("Patient checked in for Dr. X, token 14") | **no diagnosis or clinical detail.** Patient name only if the clinic setting allows it (default off: "Token 14 checked in") |
| Email / SMS | patients | booking confirmation, reminder, reschedule/cancel | appointment logistics only, **never clinical content** |
Push recipients are resolved through the same visibility predicate as the panel. Patient
messages go through a `notification_outbox` table: rows are written in the same transaction
as the event and a cron worker sends them with retries. A `(appointment_id, kind)` unique
key guarantees exactly-once delivery, generalising HRestroSewa's `report_deliveries`.

## 10. Email architecture
One platform sender over **Resend (HTTPS)**, with SMTP only as a dev fallback. HRestroSewa
found that its VPS provider drops outbound SMTP. The sender name can be "<Clinic> via
CMediSewa". Templates are code (React Email or plain HTML), and PDFs come from pdf-lib. The
mailer never throws, retries with backoff, and logs each outcome to the outbox.

## 11. PWA strategy
The staff dashboard is installable. The service worker posture is copied from HRestroSewa:
**cache nothing dynamic**, cache only the offline page and static assets, never intercept
authenticated or API responses, and serve `sw.js` with `no-store`. `OfflineGate` blocks
writes while offline. Logout clears the Cache Storage and any IndexedDB. **No patient data is
stored at rest on the device** (D-21). The public site is a normal responsive site, not a PWA,
in the MVP.

## 12. Deployment architecture
App container + **its own** self-hosted Supabase stack, both managed by Coolify. It does not
share HRestroSewa's database (D-15). Hosting location is pending (P-02, P-09). The Postgres
port is never public, and Kong `/pg` is restricted. Environments are dev, staging and prod.
Details are in `deployment.md`.

## 13. Folder structure (planned; created in Phase 0)
```
app/
  (marketing)/  (public)/c/[slug]/  (auth)/  (clinic)/clinic/  (superadmin)/superadmin/
  actions/            clinic-scoped actions, one module per domain
  actions/public/     the ONLY actions the patient site may call
  api/realtime  api/cron/*  api/push/action  api/files/[id]
lib/
  auth/  permissions.ts  care-access.ts  audit.ts  clinic-time.ts  scheduling/  billing/
  notify/  realtime/  supabase/  cache/  email/  pdf/
components/ui/  components/pwa/  components/branding/
supabase/migrations/  supabase/cron/
scripts/migrate.mjs  scripts/lib/
```

## 14. Patterns reused from HRestroSewa (summary)
Server-only data access with an RLS deny-all backstop · middleware that only redirects ·
live membership check · permission sections, levels and presets, with navigation derived
from them · record-level visibility predicate (assignment → care relationship) · topic-only
SSE realtime · push resolved through the same predicate and sent after the response ·
Resend mailer · pg_cron → secret-gated routes · exactly-once delivery table · derived values
over stored ones · single time module · migrations that own grants · guarded migration
runner. The full matrix is in `../docs/hrestrosewa-reference.md` §2.
