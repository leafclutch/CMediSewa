# Database (proposed)

Status: **Recommendation. Not final.** This is the conceptual model derived from the
workflows in `workflows.md`. Exact DDL is written in Phase 0/1 migrations and reviewed
against the workflows before it's applied. Open items are marked **[OPEN]** and cross-referenced
to `decisions.md`.

## 1. Principles
1. **Postgres on self-hosted Supabase. Shared schema. `clinic_id` on every tenant-owned row** (D-03).
2. **Composite tenant keys.** Every tenant table has `unique (clinic_id, id)`, and every
   reference to it is `foreign key (clinic_id, x_id) references x (clinic_id, id)`. A
   cross-tenant reference is then a constraint violation, not a code-review question.
3. **Access posture set in migration 0001** (D-18): RLS enabled on every table, `anon` and
   `authenticated` hold no grants (including default privileges for both grantor roles),
   and `service_role` is granted explicitly. Every later migration that adds a table must
   also enable RLS. A CI check fails if any `public` table has RLS disabled.
4. **Clinical tables are reached only through RPCs** that take `p_actor_member_id`, check the
   care relationship in SQL, and write `audit_log` in the same transaction (D-04, D-23).
5. **Never hard-delete clinical or financial rows.** Use soft delete (`deleted_at`,
   `deleted_by`, `delete_reason`) for demographics. Finalized clinical records are
   immutable and corrected by addendum (D-12). Invoices are voided, never deleted.
6. **Derived over stored:** availability, balances, queue position and subscription status.
7. **UUID primary keys** (`gen_random_uuid()`). `created_at`/`updated_at` timestamptz in
   UTC. `created_by` references a member wherever a human acts.
8. **Money** is `numeric(12,2)` in NPR. Prices are **snapshotted** onto invoice lines.
9. Every table, key column and RPC gets a `comment on` that explains *why*, as in HRestroSewa.

## 2. Entity map
```
platform_admins            plans ──< clinic_subscriptions >── clinics
                                                                 │
             ┌───────────────┬──────────────┬───────────────┬───┴──────────┬──────────────┐
        clinic_members   departments     services      clinic_settings   audit_log   notification_outbox
          │   │  (auth user ↔ clinic)        │
          │   └── doctor_profiles ──< doctor_departments >── departments
          │            │
          │            ├──< doctor_schedules (weekly template)
          │            ├──< schedule_exceptions (leave / extra hours)
          │            └──< doctor_services >── services (fee override)
          │
 patients ──< appointments >── doctor_profiles
    │             │
    │             └── visits (check-in → checkout; token) ──< encounters
    │                                   │                       ├──< vitals
    │                                   │                       ├──< diagnoses
    │                                   │                       ├──< encounter_notes / addenda
    │                                   │                       └──< prescriptions ──< prescription_items
    │                                   └──< invoices ──< invoice_items, payments
    ├──< patient_allergies, patient_conditions (medical history)
    ├──< care_access_grants (explicit shares, break-glass)
    └──< clinical_files (private storage) [Phase 2]
 Phase 2+: lab_tests, lab_orders ──< lab_order_items ──< lab_results; medicines, dispensations
```

## 3. Platform & tenancy
- **clinics**: `id, slug (unique, [a-z0-9-]), name, legal_name, pan_number, registration_no,
  phone, email, address (province, district, municipality, ward, street), geo (opt),
  logo_path, timezone default 'Asia/Kathmandu', status (active|suspended|closed),
  public_booking_enabled, created_at`.
- **clinic_settings**: one row per clinic (PK = clinic_id), split out so that frequently
  read and cached config is separate from identity. It holds `booking_window_days`,
  `min_booking_notice_minutes`, `cancellation_cutoff_hours`, `default_slot_minutes`,
  `scheduling_mode (slot|token|both)` (P-03), `push_shows_patient_name bool default false`,
  `invoice_prefix`, `invoice_pad`, receipt footer, `locale`. It is a typed table rather than
  HRestroSewa's `settings jsonb`, because booking rules are queried by SQL in the conflict
  RPCs and need constraints.
- **plans**: `code, name, price_npr, billing_cycle_days, max_doctors, max_staff, features
  text[]` (e.g. `lab`, `pharmacy`, `sms`). Managed by the Super Admin.
- **clinic_subscriptions**: `clinic_id, plan_id, starts_on, ends_on, status, amount_paid,
  payment_ref, recorded_by`. Append-only history. The current state is derived from the
  latest row covering today. **[OPEN]** Manual payment recording only in the MVP (P-12, P-14).
- **platform_admins**: `auth_user_id`, MFA required. Separate from `clinic_members`, as
  HRestroSewa's `super_admins` is separate from `restaurant_users`.

## 4. Users, roles, staff
- **clinic_members**: `id, clinic_id, auth_user_id (nullable until invited user accepts),
  role (clinic_admin|clinic_staff), staff_type (doctor|nurse|receptionist|lab_technician|
  pharmacist|accountant|other), display_name, phone, email, permissions text[] default '{}',
  is_active, deleted_at`. Constraint `unique (clinic_id, auth_user_id)`. One Auth user may
  belong to many clinics (D-24), and the MVP UI picks the single active membership.
- **doctor_profiles**: `member_id (PK/FK, clinic-scoped), nmc_registration_no
  (required for doctors; [OPEN] format/verification), qualification, specialty, bio,
  photo_path, consultation_fee_default, is_public (listed on website), public_slug`.
  Unique `(clinic_id, nmc_registration_no)`.
- **departments**: `clinic_id, name, description, is_active, sort_order`; unique
  `(clinic_id, lower(name))`.
- **doctor_departments**: M2M.
- **services**: `clinic_id, department_id (opt), name, kind (consultation|procedure|
  lab|other), price_npr, duration_minutes, is_public, is_active, tax_category`
  **[OPEN]** VAT treatment (P-08).
- **doctor_services**: M2M with an optional `price_override`.

## 5. Scheduling & appointments
- **doctor_schedules**: weekly template rows `(member_id, weekday 0-6, start_time,
  end_time, slot_minutes, max_tokens (opt), location_note, effective_from, effective_to)`.
  A CHECK ensures `start < end`, and an exclusion constraint prevents overlapping template
  windows for the same doctor on the same weekday.
- **schedule_exceptions**: `(member_id, date, kind (leave|extra|modified), start_time,
  end_time, reason)`. Leave blocks slots, and existing appointments in that range are
  flagged for rescheduling (W-11).
- **Slots are derived, not stored**: template + exceptions − booked appointments, computed
  in a SQL function `available_slots(clinic, doctor, from, to)`. **[OPEN]** If slot
  generation turns out slow it can be materialised later. The expected scale (≤15 doctors)
  doesn't need it.
- **appointments**: `id, clinic_id, patient_id, doctor_member_id, service_id (opt),
  department_id (opt), scheduled_range tstzrange (null for pure token walk-ins), token_no
  (opt), source (online|phone|walk_in|follow_up), status, booked_by_member_id (null when
  online), reason_for_visit (short, patient-entered, treated as PHI), manage_token_hash,
  cancelled_at, cancel_reason, cancelled_by_kind (patient|staff|system),
  rescheduled_from_id`.
  - **Status machine:** `requested → confirmed → checked_in → in_consultation → completed`.
    Exits are `cancelled` and `no_show`. Transitions happen only through RPCs, and a trigger
    rejects illegal transitions.
  - **No double booking:** `exclude using gist (clinic_id with =, doctor_member_id with =,
    scheduled_range with &&) where (status in ('requested','confirmed','checked_in',
    'in_consultation'))` (btree_gist). The database rejects a race between two patients for
    the same slot (D-11).
  - **Patient self-overlap:** a second exclusion on `(clinic_id, patient_id, scheduled_range)`
    for active statuses.
  - **Abuse:** booking RPC enforces "max N future online bookings per phone per clinic".
- **visits**: `id, clinic_id, patient_id, appointment_id (opt, unique), visit_date (clinic
  local date), token_no, checked_in_at, checked_in_by, checked_out_at, status`. Constraint
  `unique (clinic_id, doctor_member_id, visit_date, token_no)`. Tokens are assigned atomically
  by the check-in RPC from a per-doctor-per-day counter row (`token_counters`) with
  `select … for update`.

## 6. Patients
- **patients**: `id, clinic_id, mrn (unique per clinic, human-readable, e.g. CM-2083-000123,
  stamped by trigger), full_name, full_name_ne (opt, Devanagari), sex, date_of_birth (opt) or
  age_years_at_registration (many patients know age, not DOB: **Assumption**), phone,
  alt_phone, email (opt), address fields, guardian_name/relation (minors), national_id_no
  (opt, **[OPEN]** collect at all? data minimisation), blood_group (opt), created_via
  (online|desk), deleted_at`.
  - Per-clinic identity: the same human at two clinics is two rows (D-20). There is no
    cross-clinic patient lookup.
  - Duplicate detection: on registration and online booking, match on `(clinic_id,
    phone, normalised name, dob/age)` and prompt reception to merge. **[OPEN]** A merge
    workflow is Phase 2 (merges must be audited and reversible).
- **patient_allergies**: `patient_id, allergen, reaction, severity, recorded_by,
  recorded_at, status`. Shown on every prescription screen. It is part of the MVP, because
  prescriptions need it for safety.
- **patient_conditions** (medical history): `patient_id, condition, since, status
  (active|resolved), notes`. MVP-lite (free text).
- **consents** **[OPEN]**: `patient_id, kind (data_processing|sms|…), given_at, method,
  withdrawn_at`. Needed for privacy-law alignment (P-10, `security.md` §10).

## 7. Clinical records
- **encounters**: `id, clinic_id, visit_id, patient_id, doctor_member_id, started_at,
  finalized_at, finalized_by, status (draft|finalized|amended)`.
  - `chief_complaint, history, examination, assessment, plan, follow_up_on (date),
    advice` are text columns on the encounter. **Recommendation:** start structured-lite
    (fixed sections) and avoid a form builder.
  - **Immutability:** after `finalized_at`, a trigger rejects updates to clinical columns.
    Corrections go into **encounter_addenda** `(encounter_id, text, author, created_at)`.
- **vitals**: `encounter_id or visit_id, bp_systolic, bp_diastolic, pulse, temp_c,
  spo2, resp_rate, weight_kg, height_cm, recorded_by, recorded_at`, with CHECK ranges for
  plausible values.
- **diagnoses**: `encounter_id, text, icd10_code (opt; P-15), is_primary`.
- **prescriptions**: `id, clinic_id, encounter_id, patient_id, doctor_member_id,
  issued_at, rx_number (per clinic sequence), status (draft|issued|cancelled), notes`.
  Issued prescriptions are immutable. Cancelling one requires a reason and is audited.
- **prescription_items**: `medicine_name (free text MVP; Phase 3 links to medicines),
  strength, dosage_form, dose, frequency, route, duration_days, quantity, instructions`.
- **care_access_grants**: `patient_id, member_id, kind (referral|share|break_glass),
  reason, granted_by, starts_at, expires_at`. See `permissions.md` §5.

## 8. Billing & payments
- **invoices**: `id, clinic_id, visit_id (opt), patient_id, invoice_number (stamped on
  issue, per clinic per Nepali fiscal year, sequential, never reused; D-22),
  fiscal_year (e.g. '2083/84'), status (draft|issued|void), subtotal, discount,
  tax, total, issued_at, issued_by, void_reason, voided_by`.
  - Balance = total − sum(payments − refunds). It is **derived**.
- **invoice_items**: `service_id (opt), description, qty, unit_price (snapshot),
  discount, tax_category, line_total`.
- **payments**: `invoice_id, method (cash|online|card), amount, reference (e.g. wallet
  txn id), received_by, received_at`. A refund is a negative row with a reason
  (HRestroSewa's signed-amount pattern), so a refund needs no second table.
- **[OPEN]** IRD billing-software requirements, VAT exemption for health services,
  abbreviated tax invoice rules and fiscal-year rollover (P-08). The schema keeps
  `tax_category` and `fiscal_year` so whatever the answer turns out to be, the data
  model can support it.

## 9. Audit logging
- **audit_log** (append-only, D-23): `id bigserial, clinic_id (null for platform events),
  actor_kind (member|platform_admin|patient_token|system), actor_id, action (e.g.
  patient.read, encounter.read, encounter.finalize, prescription.issue, invoice.void,
  member.permissions_change, break_glass.open, export.csv), target_type, target_id,
  patient_id (denormalised for "who looked at this patient" queries), outcome
  (success|denied|error), reason, ip, user_agent, detail jsonb (before/after for writes;
  never full clinical text), created_at`.
  - No UPDATE or DELETE privilege for any role, including service_role: a trigger raises
    on update/delete. This is enforced in the migration.
  - Partition by month once volume warrants it **[OPEN]**. Retention is longer than or equal
    to medical-record retention (P-10).
- **Reads are audited:** opening a patient's clinical chart, viewing or printing a
  prescription, and exporting are logged. Listing appointments (which shows names and times
  only) is not, to keep the volume sensible. The boundary is defined in `security.md` §6.

## 10. Notifications
- **notifications**: the in-app staff panel (`clinic_id, recipient_member_id or audience,
  topic, title, body, link, read_at`).
- **push_subscriptions**: `member_id, endpoint, keys, ua, created_at`.
- **notification_outbox**: `clinic_id, channel (email|sms|push), kind
  (booking_confirmed|reminder_24h|rescheduled|cancelled|…), appointment_id, recipient,
  payload jsonb (PHI-minimal), status, attempts, last_error, send_after, sent_at`, with
  `unique (appointment_id, kind, channel)` for exactly-once delivery.

## 11. Key constraints & indexes (proposed)
| Purpose | Constraint / index |
|---|---|
| Tenant integrity | `unique (clinic_id, id)` on every tenant table, composite FKs |
| No doctor double booking | GiST exclusion on `(clinic_id, doctor, scheduled_range)` for active statuses |
| No patient self-overlap | GiST exclusion on `(clinic_id, patient, scheduled_range)` for active statuses |
| One token per doctor per day | `unique (clinic_id, doctor, visit_date, token_no)` |
| MRN, invoice no., rx no. | `unique (clinic_id, mrn)`, `unique (clinic_id, fiscal_year, invoice_number)`, `unique (clinic_id, rx_number)` |
| Slug / names | `unique (slug)`, `unique (clinic_id, lower(name))` for departments and services |
| Lookup | `patients (clinic_id, phone)`, `patients (clinic_id, lower(full_name) text_pattern_ops)` or trigram, `appointments (clinic_id, doctor, lower(scheduled_range))`, `visits (clinic_id, visit_date)`, `audit_log (clinic_id, patient_id, created_at)` |
| Immutability | triggers on `encounters` (post-finalize), `prescriptions` (post-issue), `invoices` (post-issue except void), `audit_log` (always) |
HRestroSewa's lesson that performance is about round-trips still holds, so index only for real
access paths. The composite-uniqueness indexes are required anyway.

## 12. Assumptions & unresolved decisions
- **Assumption:** single-location clinics in the MVP, so no `branch_id`. Adding it later means
  one nullable column plus backfill on scheduling tables.
- **Assumption:** prescriptions use free-text medicine names until the pharmacy module.
- **[OPEN] P-03:** slot vs token scheduling (the model supports both).
- **[OPEN] P-08:** tax/IRD invoice rules.
- **[OPEN] P-10:** retention periods and what "deletion" means for clinical data.
- **[OPEN] P-15:** ICD-10 coding.
- **[OPEN] P-16:** whether multi-clinic membership is exposed in the MVP UI.
- **[OPEN]** Whether `national_id_no` is collected at all.
