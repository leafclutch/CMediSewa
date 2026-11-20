# Decisions

Append-only. Never delete an entry; add a follow-up that supersedes it. Every entry has a
status:
- **Proposed**: the recommendation from the 2026-11-20 architecture analysis, awaiting the
  owner's review.
- **Accepted**: confirmed by the owner.
- **Superseded**: replaced by a later entry.
`P-xx` entries are **pending decisions**, each with alternatives and a recommended default.

---

## Accepted / proposed decisions

**D-01 Technology stack. Proposed (2026-11-20).**
Next.js App Router + React + TypeScript (strict) + Tailwind v4 + radix-ui + lucide; Supabase
(Postgres, Auth, Storage) self-hosted; web-push; SSE; Resend; pdf-lib; `pg`.
*Reason:* HRestroSewa runs this exact stack in production for a similar Nepali SMB audience.
The team knows it, its traps are documented, and its components can be copied. Server
Actions as the API keep data access server-only, which the security model depends on.
*Alternatives:* a separate API (NestJS/Express) + SPA was rejected because it doubles the
surface area and auth plumbing for no MVP benefit. Laravel/Django were rejected because
they would lose all reuse from HRestroSewa. Firebase was rejected because its relational
integrity is weak and medical records need constraints.

**D-02 Copy reusable code; don't share a package. Proposed.**
*Reason:* the shared pieces are small (about 20 files). A monorepo or package would couple
CMediSewa's releases to a production POS. *Revisit* if a third product appears.

**D-03 Multi-tenancy: shared DB, shared schema, `clinic_id` + composite FKs. Proposed.**
*Reason:* proven in HRestroSewa, cheap to run and simple to migrate. Composite
`(clinic_id, id)` FKs add a database-level guarantee HRestroSewa lacks.
*Alternatives:* schema-per-tenant (migration fan-out, pooling pain) and DB-per-tenant (ops
cost far beyond small clinics) were both rejected for now. DB-per-tenant could be an
enterprise tier later.

**D-04 Data access: server-only via service_role, RLS deny-all backstop, clinical access via
audited RPCs. Proposed.**
*Reason:* the anon key is public, so the browser must have no table access. Putting the
care-relationship check *and* the audit insert in the same SQL transaction makes "read
without audit" impossible. *Alternative:* per-user RLS with JWT claims as the main layer was
rejected for the MVP (`security.md` §6 has the full reasoning) and kept as an option for
future direct-client features.

**D-05 Staff authentication: email/username + strong password + MFA, no PIN as primary.
Proposed.**
*Reason:* HRestroSewa's 4-digit PIN fits a POS floor but is too weak for medical records.
MFA scope is P-11. A quick-unlock PIN for shared devices *on top of* a real session is P-06.

**D-06 The Clinic Admin does not implicitly access clinical content. Proposed.**
*Reason:* in HRestroSewa the admin bypasses everything, but in a clinic a non-clinical
manager reading diagnoses is the main insider risk. Admin-doctors get access as doctors.
Others use audited break-glass. The owner-doctor alternative is P-07.

**D-07 One clinic dashboard (`(clinic)`) for admin and staff, gated by permission. Proposed.**
*Reason:* HRestroSewa's split `(admin)`/`(employee)` trees led to thin pages that reuse
admin clients anyway. A single tree with derived navigation is simpler.

**D-08 The Super Admin has no access to patient data. Proposed.**
*Reason:* the platform operator doesn't need PHI, and excluding it shrinks the breach
surface and simplifies the processor role. Support works by clinic-side screen sharing, or
later through clinic-approved audited support sessions.

**D-09 Realtime: Postgres LISTEN/NOTIFY → SSE, topic-only. Proposed.**
*Reason:* proven in HRestroSewa, and it keeps RLS closed to the browser. Clients refetch
through permission-checked actions, so no data travels over the stream.

**D-10 Notifications: outbox + cron dispatcher; PHI-minimal content on every external
channel. Proposed.**
*Reason:* the outbox is written in the same transaction as the event, giving exactly-once
delivery and retries (a generalisation of HRestroSewa's `report_deliveries`). Push, SMS and
email cross third-party infrastructure, so they never carry clinical content.

**D-11 Double booking prevented by GiST exclusion constraints. Proposed.**
*Reason:* the database guarantees it under concurrency, whereas application checks can race.
Uses `btree_gist`.

**D-12 Clinical records are immutable after finalize/issue; corrections are addenda or
cancellations. Proposed.**
*Reason:* medico-legal integrity. The same "edit in place + audit" pattern as HRestroSewa's
Security PIN is **not** appropriate for clinical content.

**D-13 Public site URL: `/c/[slug]` path in the MVP; subdomains in P2; custom domains later.
Proposed.**
*Reason:* one TLS certificate, no wildcard DNS work for the MVP, and the same pattern as
HRestroSewa. Depends on P-01 for the actual domain.

**D-14 Time: UTC storage, clinic timezone (Asia/Kathmandu), AD + BS display, one time
module. Proposed.**
*Reason:* HRestroSewa's `business-day.ts` lesson is that scattered date maths breaks at day
boundaries. The BS conversion library is **[OPEN]** and must be evaluated for correctness
across the supported year range.

**D-15 Deployment: Coolify + self-hosted Supabase, stack separate from HRestroSewa; Kong `/pg`
blocked publicly. Proposed.**
*Reason:* reuse of proven ops, plus isolation from the POS. Blocking `/pg` removes a
superuser-reaching HTTP route that HRestroSewa accepts. Tooling uses an SSH tunnel. Hosting
location is P-02/P-09.

**D-16 Pin exact dependency versions. Proposed.**
*Reason:* HRestroSewa's `latest` ranges make builds non-reproducible. Medical software needs
reproducible, reviewable upgrades.

**D-17 Email through Resend (HTTPS); SMTP only as a dev fallback. Proposed.**
*Reason:* HRestroSewa confirmed its VPS provider drops outbound SMTP.

**D-18 Migration 0001 sets the access posture before any table exists. Proposed.**
*Reason:* HRestroSewa found that hosted and self-hosted Supabase default grants differ in
opposite directions. Making the posture explicit in migrations is the only reproducible
setup.

**D-19 MVP onboarding is Super-Admin-provisioned (invite-only); self-signup in P2. Proposed.**
*Reason:* the first clinics need hand-holding and verification. Self-signup brings fraud
and abuse surface.

**D-20 Patients are per-clinic records; no cross-clinic patient identity. Proposed.**
*Reason:* each clinic is the data controller for its own patients. A cross-clinic identity
would need consent and legal grounds that haven't been established. A patient portal (P2)
can link the same phone across clinics later, with consent.

**D-21 No patient data stored at rest on client devices. Proposed.**
*Reason:* shared reception PCs and personal phones. HRestroSewa's no-dynamic-cache service
worker posture already fits.

**D-22 Invoice numbers sequential per clinic per Nepali fiscal year, stamped on issue, never
reused. Proposed.**
*Reason:* HRestroSewa's trigger-stamped numbering is proven. Fiscal-year scoping is the
expected Nepali practice (**Assumption**, verify under P-08).

**D-23 Append-only `audit_log`, including reads of clinical data. Proposed.**
*Reason:* "who looked at this patient" must be answerable. A trigger forbids update/delete
for every role.

**D-24 `clinic_members` decouples Auth users from clinics (many-to-many-ready). Proposed.**
*Reason:* visiting doctors commonly serve several clinics (**Assumption**). Building the
table now costs nothing, whereas retrofitting it later touches auth everywhere. Whether
the MVP UI exposes it is P-16.

**D-25 Storage: public bucket for logos and doctor photos; private bucket + short-TTL signed
URLs (after care check + audit) for clinical files. Proposed.**
*Reason:* HRestroSewa has only public buckets, which is fine for menus and unacceptable for
clinical documents.

---

## Pending decisions

| ID | Question | Alternatives | Recommended default | Blocking? |
|---|---|---|---|---|
| **P-01** | Platform domain & URL scheme | `.com` / `.com.np`; path vs subdomain for clinics | buy the domain now; path `/c/[slug]` in MVP (D-13) | Phase 0 |
| **P-02** | Hosting location | dedicated VPS / same VPS as HRestroSewa / Nepal DC / hosted cloud | dedicated VPS (`deployment.md` §3, option A) unless P-09 requires Nepal | Phase 1 step 13 (prod) |
| **P-03** | Scheduling mode | time slots / daily tokens / both per doctor | **both**, configurable per doctor schedule | Phase 1 step 4 |
| **P-04** | Online booking verification | none / SMS OTP / manual confirmation | CAPTCHA + rate limits + optional manual confirmation in MVP; OTP once P-05 done | no |
| **P-05** | SMS / messaging provider | Nepali SMS aggregators (e.g. Sparrow SMS, Aakash SMS: verify availability/pricing), Viber/WhatsApp business | evaluate cost, DLR reliability, sender-ID rules; decide before P2 | P2 |
| **P-06** | Shared reception device sign-in | individual login each time / quick-switch PIN over a device session / idle lock | idle lock + re-auth in MVP; quick-switch in P2 | no |
| **P-07** | Admin clinical access & doctor history window | strict (D-06) / clinic setting "admin clinical read (audited)" / limit C2 to N months | strict + C2 unlimited (audited) | Phase 1 step 9 |
| **P-08** | Tax & IRD billing compliance; Rx format rules | see `security.md` §10 | keep `tax_category`, `fiscal_year` fields; get counsel opinion | before go-live |
| **P-09** | Data residency | foreign VPS / Nepal hosting | depends on counsel reading of privacy & cyber policy | before go-live |
| **P-10** | Retention & deletion periods; consent text | — | soft delete only; placeholder 10 years; counsel to set | before go-live |
| **P-11** | MFA scope | Super Admin only / + Clinic Admin / + doctors / everyone | mandatory: Super Admin + Clinic Admin + doctors; optional others | Phase 1 step 1 |
| **P-12** | Online payments (subscriptions & patients) | manual recording / eSewa / Khalti / Fonepay | manual in MVP; gateway in P2 | P2 |
| **P-13** | Language | English only / bilingual from MVP | i18n-ready MVP, English UI; Nepali public site P2 | no |
| **P-14** | Plans & pricing | per clinic / per doctor / tiered features | per-clinic tiers with doctor caps (schema supports either) | before go-live |
| **P-15** | Diagnosis coding | free text / ICD-10 / ICD-11 | free text + optional ICD-10 field | no |
| **P-16** | Multi-clinic membership in MVP UI | hidden / clinic switcher | hidden in MVP (model supports it, D-24) | no |
