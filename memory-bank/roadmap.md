# Roadmap

Status: **Recommendation.** No dates are committed. Phases are gated by exit criteria, not
by a calendar. Module IDs are listed in `features.md`.

## Dependency graph (MVP)
```
Phase 0 foundation
   └─► M-03 Auth ─► M-01 Clinics (Super Admin) ─► M-02 Subscriptions-lite
                         └─► M-05 Settings ─► M-04 Staff & permissions
                                               └─► M-06 Depts & services ─► M-07 Doctors ─► M-08 Schedules
   M-19 Audit (built alongside M-03; every later module writes to it)
   M-09 Patients ─────────────────────────────────────────────┐
   M-08 + M-09 ─► M-10 Appointments ─► M-11 Public site & booking
                        └─► M-12 Check-in & queue ─► M-13 Consultation (+M-15) ─► M-14 Prescriptions
                                     └─────────────────────────► M-16 Billing
   M-10/M-12 ─► M-17 Notifications (in-app, push, email)
   M-10/M-16 ─► M-18 Reports
```

## Phase 0: Foundation (no clinic features)
1. Resolve the **blocking** pending decisions: P-01 (domain), P-02 (hosting), P-03
   (scheduling mode), P-07 (admin clinical access), P-11 (MFA scope). Others can wait
   (`decisions.md`).
2. Scaffold the Next.js app (pinned versions, TS strict, ESLint, Tailwind v4) and copy the
   HRestroSewa components listed in `../docs/hrestrosewa-reference.md` §2B, adapted.
3. Supabase local dev + **migration 0001: access posture** (RLS defaults, revoke
   anon/authenticated, explicit service_role grants, default privileges for both grantors).
   Migration runner with dev/staging/prod interlocks.
4. CI: typecheck, lint, unit tests, migration replay on an empty DB, RLS/grant posture check,
   and a service-key-in-bundle scan.
5. Design tokens and the base layout shells for each route group (`ui-ux.md`).
6. Staging environment on Coolify with synthetic data.
**Exit:** an empty app deploys to staging through CI. The posture check passes on a fresh DB.

## Phase 1: MVP (the build order follows dependencies)
| Step | Modules | Technical milestone | Testing milestone |
|---|---|---|---|
| 1 | M-03, M-19 | Auth (invite, password, MFA, lockout), guards, live membership check, `audit_log` + append-only trigger | auth E2E; lockout; deactivated user loses access on the next request |
| 2 | M-01, M-02 | Super Admin: clinics CRUD, admin invite, plans, subscription periods, expiry cron | Super Admin can't reach any clinic route or patient table (negative tests) |
| 3 | M-05, M-04 | Clinic settings, staff invite, permission sections + presets, staff-type gating, derived nav | permission matrix tests (each preset × each route/action) |
| 4 | M-06, M-07, M-08 | Departments, services, doctors (NMC no.), schedule templates, exceptions, `available_slots()` | slot derivation unit tests (exceptions, leave, effective dating, timezone/BS edges) |
| 5 | M-09 | Patients, MRN trigger, search, duplicate detection | **two-clinic isolation test suite** established here and extended every later step |
| 6 | M-10 | Appointment RPCs (book, confirm, cancel, reschedule, no-show), GiST exclusion, status-machine trigger | **concurrency test:** N parallel bookings for one slot gives exactly 1 success |
| 7 | M-11 | Public site, booking flow, manage link, CAPTCHA, rate limits | public-contract test (no MRN or clinical fields); abuse-limit tests |
| 8 | M-12 | Check-in RPC, tokens, queue board (SSE realtime bus) | token uniqueness under concurrency; SSE fallback poll |
| 9 | M-13, M-15 | Encounter RPCs, care predicate in SQL, chart read audit, finalize immutability, addenda | **care-relationship matrix tests** (C1–C6), denied reads audited |
| 10 | M-14 | Rx issue, allergy check, PDF/print | Rx immutability; print audit; PDF snapshot test |
| 11 | M-16 | Invoices, fiscal-year numbering, payments, void, refund, receipts | numbering never reused or skipped; balance derivation; fiscal-year rollover |
| 12 | M-17 | Notification outbox, email (Resend), push (VAPID), dispatch cron | exactly-once; PHI-free payload assertions |
| 13 | M-18 | Basic reports | derived totals equal the source rows |
**Deployment milestones:**
- **D1** (after step 3): staging is live with auth, clinics and staff for internal review.
- **D2** (after step 7): staging demo of public booking for a pilot clinic.
- **D3** (after step 13): production stack provisioned (P-02), backups + PITR + off-site
  verified, **restore drill passed**, monitoring on, and security review done (§Exit).
- **D4:** pilot clinic go-live (one clinic, real data), with a 2–4 week stabilisation period.

**MVP exit criteria:** the `project.md` MVP definition passes on staging end to end; the
isolation, care-access, concurrency and PHI-leak test suites are green; the pre-launch
legal items in `security.md` §10 have at least a counsel opinion on consent text, retention
and invoicing; and a restore drill is documented.

## Phase 2: Useful additions (order is a suggestion, driven by pilot feedback)
1. M-22 SMS reminders (after P-05) and M-26 follow-ups & referrals.
2. M-20 Laboratory (catalogue, orders, results, report PDF) and M-21 clinical files
   (private bucket, signed URLs).
3. M-27 Daily summary email (HRestroSewa reporting service shape).
4. M-24 Online payments (eSewa/Khalti/Fonepay, after a gateway decision).
5. M-23 Patient portal (phone OTP) and M-28 patient merge.
6. M-25 Self-service clinic signup and trial. Subdomain public sites. Nepali UI.
7. Queue TV display, shared-device quick switch (P-06), credit/due billing.

## Future expansion
M-29 Pharmacy & inventory (reuse HRestroSewa's *derived stock* pattern, with batches and
expiry) · M-30 Insurance (HIB/SSF) claims · M-31 Multi-branch · M-32 Telemedicine ·
M-33 FHIR / LIS integration · M-34 Custom domains · analytics and benchmarking across
clinics (aggregate, anonymised, opt-in).

## Risks to schedule
- Legal answers (P-08, P-09, P-10) may change invoice format or hosting late. Keep tax fields
  and the hosting choice flexible until the answers arrive.
- Care-access rules are the most likely to be renegotiated with real doctors. They live in
  one predicate (TS + SQL) so a change is one place.
- SMS cost and delivery in Nepal are unknown (P-05). Email-only reminders may underperform
  with patients who don't use email.
