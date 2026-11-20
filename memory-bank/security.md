# Security & Privacy

Status: **Recommendation.** CMediSewa makes **no compliance claim** of any kind. §10 lists
legal areas that need verification by qualified Nepali counsel before launch.

## 1. Threat model (what we defend against)
| Threat | Example | Primary controls |
|---|---|---|
| Cross-tenant leak | bug returns clinic B's patients to clinic A | `clinic_id` from live membership, composite FKs, scoped data helper, tests (§4) |
| Curious insider | receptionist reads a neighbour's diagnosis | capability + care relationship, read audit, admin review (§5, §6) |
| Compromised staff account | phished password | MFA, lockout, idle timeout, session revocation, anomaly alerts (§2) |
| Public-booking abuse | slot squatting, spam, enumeration | CAPTCHA, rate limits, per-phone caps, hashed manage tokens, no enumeration (§8) |
| Leaked anon key | the key is public by design | zero grants to `anon`/`authenticated` (§4) |
| Server compromise / DB exposure | exposed Postgres or `/pg` route | no public DB port, `/pg` blocked, SSH key-only, patching (§7) |
| Data loss | disk failure, bad migration, ransomware | off-site encrypted backups, PITR, restore drills (§9) |
| Leak through side channels | PHI in push, SMS, logs or error reports | content rules + redaction (§7) |

## 2. Authentication
- Supabase Auth, **invite-only** (public signup disabled). Staff use email (or a
  clinic-scoped username mapped to an internal email) plus a password of 10 characters or
  more, checked against a common-password list.
- **MFA (TOTP):** mandatory for Super Admins. Strongly recommended, and mandatory if P-11
  decides so, for Clinic Admins and doctors. Re-authentication is required before
  break-glass, before staff-permission changes and before exports.
- **Lockout:** HRestroSewa's DB-clock lockout (N failures → timed lock), applied to password
  attempts and MFA codes. The admin can unlock, and the unlock is audited.
- **Sessions:** HttpOnly, Secure, SameSite=Lax cookies (via `@supabase/ssr`). Access-token
  TTL about 1h with refresh rotation. **Idle timeout** of around 15 minutes on clinical
  screens (a lock screen that needs re-entry; P-06 decides between a PIN and a password).
  Logout is local scope. "Sign out everywhere" is a deliberate action, available to the
  user and to the admin for any member.
- **Deactivation is immediate:** the membership row is read live on every request (an
  HRestroSewa pattern).
- Local JWT verification via `getClaims()` uses asymmetric signing keys and falls back
  to the network on error. It never treats an error as "signed out".

## 3. Authorization
Three independent checks: tenant, capability, record (`permissions.md` §1). They are
enforced **server-side in every page and action**. Clinical data is reachable only through
RPCs that check the care relationship in SQL, so one forgotten check in TypeScript cannot
expose a chart. The Clinic Admin doesn't bypass clinical rules and the Super Admin has no
clinical access (D-06, D-08).

## 4. Tenant isolation
1. `clinic_id` comes **only** from the server-resolved membership, never from a form field,
   URL or cookie. Actions that receive an id (patient, appointment) verify that it belongs
   to `ctx.clinic_id` by querying with both.
2. **Composite foreign keys** make cross-tenant references impossible in the DB.
3. The `db(ctx)` helper injects `clinic_id` into clinic-scoped queries.
4. Tenant-scoped caches include `clinic_id` in both the key and the tag (`tenantCache`).
   **PHI is never cached.**
5. **The browser never queries the DB.** `anon`/`authenticated` hold no grants. RLS is on
   for every table as a deny-all backstop. CI fails on any table with RLS off or any grant
   to those roles.
6. **Automated isolation tests:** for every action and RPC, a test seeds two clinics and
   asserts that clinic A's member can't read, write or reference clinic B's rows. This is a
   gate in the MVP's definition of done.

## 5. Patient data protection
- **Minimise:** collect only what the workflow needs. National ID is not collected by default
  (**[OPEN]**). Reason-for-visit is optional.
- **Classify:** *clinical* (encounters, notes, diagnoses, Rx, allergies, conditions, lab,
  files), *personal* (demographics, contact) and *operational* (appointments, tokens,
  invoices). Each class has its own access rules (`permissions.md` §5).
- **Immutability:** finalized encounters, issued prescriptions and issued invoices are
  frozen by triggers. Corrections are addenda, cancellations or voids, each with an actor and
  a reason.
- **No PHI on devices:** the service worker caches nothing dynamic. Logout clears caches. No
  localStorage or IndexedDB for patient data. Print views are rendered on demand.
- **Encryption:** TLS everywhere. Disk/volume encryption on the VPS where the provider
  supports it. Backups are encrypted before leaving the host. Column-level encryption of
  clinical text is **not** proposed for the MVP: the DB is server-only, and it would break
  search. Revisit if counsel or a risk review asks for it.
- **Exports:** admin-only for clinical content, audited, and watermarked with the
  exporter's name and time.

## 6. Supabase RLS strategy
**Recommended (D-04): server-only access + deny-all RLS + clinical RPCs.**
- Every table has RLS enabled and no policies for `anon`/`authenticated` (they also hold no
  grants). All access goes through `service_role` on the server.
- Clinical reads and writes go through `SECURITY INVOKER` RPCs executed by `service_role`
  that receive `p_actor_member_id` and `p_clinic_id` (taken from the verified server
  context), evaluate the **care predicate** in SQL and insert the `audit_log` row in the
  same transaction. A denied read is also audited (`outcome = denied`).
- **Why not per-user RLS (JWT claims + policies) as the main layer?** It would require
  exposing tables to the `authenticated` role (so any XSS or token theft becomes direct
  table access), custom access-token hooks for `clinic_id`/permissions, and policies
  complex enough to encode the care rules, which are hard to test and slow. HRestroSewa
  rejected browser-side access for the same reason. **Alternative kept open:** if a future
  feature needs direct client queries (e.g. a native app), introduce per-user RLS for that
  table set only, with policies mirroring the care predicate.
- **What gets audited as a read:** opening a chart, viewing an encounter, viewing or printing
  an Rx, viewing lab results, downloading a clinical file, break-glass and exports.
  Lists and searches that show demographics and appointment times only are not logged per
  row. Opening a patient's profile page is logged as `patient.profile_read`.

## 7. API security
- **Server Actions** are the API. Each re-validates input with schema validation (zod or
  valibot) and checks auth and permissions. Pure helpers are never exported from
  `"use server"` files. Next's built-in Server Action origin check stays on, with
  `allowedOrigins` limited to the app origin.
- **Cron routes:** `x-cron-secret` compared in constant time, 404 on failure, POST only.
  Secrets come from Vault for pg_cron.
- **Rate limiting:** per IP and per identity on login, booking, manage-link actions and OTP
  (P-04). Stored in Postgres (simple counter table) or in memory for one instance (one
  container in the MVP). Multi-instance later needs Redis or Postgres.
- **Headers:** CSP (no inline scripts except Next nonces; `frame-ancestors 'none'` for the
  dashboard), HSTS, `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`
  (so manage-token URLs don't leak), and `Permissions-Policy`.
- **Uploads:** MIME sniffing plus an allow-list, a size cap, uuid names, private bucket for
  clinical files and image re-encoding for public photos.
- **Secrets:** server-only modules (`server-only` import). The service key is never in client
  bundles. CI scans the bundle for it.
- **PHI-free side channels:** push payloads, SMS, email, logs, error reports, analytics and
  URLs (no names or phones in query strings) all exclude clinical content. There is no
  third-party analytics on dashboard pages. The public site may use privacy-friendly
  analytics without cookies.
- **Infrastructure:** no public Postgres port, Kong `/pg` and Studio blocked publicly, SSH
  key-only, automatic patching, Docker images updated monthly.

## 8. Secure appointment booking
- **Inputs:** validated server-side (phone `+977` format, name length, date windows).
- **Bots and spam:** CAPTCHA on submit, per-IP and per-phone rate limits, and at most N
  active future online bookings per phone per clinic (default 2). Repeated no-shows lower
  that limit (W-11).
- **Optional OTP** (P-04): SMS OTP to confirm phone ownership before the booking is
  confirmed. Until it exists, online bookings may be configured as `requested` (manual
  confirmation) for clinics that want it.
- **No enumeration:** public endpoints never reveal whether a phone or patient already
  exists. Matching happens server-side and the response is identical either way.
- **Manage tokens:** 128-bit random, stored as an HMAC-SHA256 hash with a pepper, scoped to
  one appointment, expiring after the appointment date plus N days, and rotated on reschedule.
  The page shows only that appointment's logistics.
- **Conflicts:** GiST exclusion constraints make double booking impossible under concurrency.
  The RPC returns `SLOT_TAKEN` and the UI offers the nearest alternatives.
- **Public data contract:** `app/actions/public/*` returns explicit allow-listed types. A
  test asserts that no public response contains an MRN, a patient id other than the
  caller's own booking, or any clinical field.

## 9. Backup & recovery
See `deployment.md` §9: nightly verified dumps, encrypted off-site copies, WAL-based PITR
(RPO ≤ 15 min, RTO ≤ 4 h targets) and monthly restore drills with parity checks.
Disaster runbook: restore order (DB → storage → app), DNS cutover, key rotation if a
compromise is suspected, and clinic communication.

## 10. Privacy & compliance, to verify (not claimed)
These Nepal-specific areas were identified as relevant. **None has been verified for
CMediSewa.** Each needs confirmation from current official texts and/or qualified Nepali
legal counsel before production launch (tracked as P-08, P-09, P-10):

| Area | Instrument (to verify current text) | Questions for counsel |
|---|---|---|
| Constitutional privacy | Constitution of Nepal 2072 (2015), Art. 28 | baseline right that the other laws implement |
| Personal data & health privacy | **Individual Privacy Act 2075 (2018)** and **Privacy Regulation 2077 (2020)** | consent requirements for health data; purpose limitation; disclosure rules; data-subject access/correction; obligations of a data processor (CMediSewa) vs controller (clinic); penalties |
| Health institutions | **Public Health Service Act 2075 (2018)** and Regulation 2077 | patient confidentiality and record duties; **medical-record retention periods**; health-facility reporting (e.g. HMIS/DHIS2) obligations that clinics may expect the system to support |
| Professional conduct | **Nepal Medical Council** Act and Code of Ethics | confidentiality; prescription content (registration no., signature); validity of electronic prescriptions/signatures |
| Electronic records & signatures | **Electronic Transactions Act 2063 (2008)** | legal standing of electronic medical records and digital signatures; unauthorised-access offences |
| Medicines | **Drugs Act 2035 (1978)**, Department of Drug Administration rules | Rx format; controlled-drug prescriptions; pharmacy module requirements (Future) |
| Tax & billing | **VAT Act 2052 (1996)** and Rules, Inland Revenue Department billing-software rules | whether health services are VAT-exempt; mandatory invoice fields; whether billing software must be IRD-registered/certified; fiscal-year numbering; real-time reporting (CBMS) applicability |
| Data residency & security | **National Cyber Security Policy 2080 (2023)** and any directives | whether health data must be hosted in Nepal (drives P-09); incident reporting duties |
| Consumer protection | Consumer Protection Act 2075 (2018) | booking/cancellation terms and fee disclosures on the public site |
| Clinic licensing | Ministry of Health and Population health-institution standards | any software or record-keeping requirements in licensing standards |

**Product guardrails until verified:** collect explicit consent at registration and booking
(text drafted with counsel); provide data-subject request handling (export/correct) as an
admin workflow; define **clinic = controller, CMediSewa = processor** in a Data Processing
Agreement; publish a privacy notice on each clinic site; **never hard-delete clinical
records** until retention rules are known (soft delete + restricted access); keep audit logs
at least as long as records.

**Data retention & deletion policy (draft, pending P-10):**
- Clinical and financial records are retained for the legally required period (unknown;
  placeholder **10 years** from the last encounter). The Clinic Admin cannot hard-delete them.
- Demographic "erasure" requests: restrict and anonymise contact fields where the law
  permits, and keep clinical content as required.
- A closed clinic's data is exported to the clinic (encrypted archive) and then retained or
  deleted according to the DPA and the law.
- Logs are kept 30–90 days. Backups follow the backup rotation, so deleted data ages out of
  backups by the end of the rotation.
- Retention jobs run by pg_cron are audited and dry-run first.
