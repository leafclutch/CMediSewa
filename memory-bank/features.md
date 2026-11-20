# Features

Module inventory with phase, purpose, dependencies, roles and data. IDs (`M-xx`) are used
in `roadmap.md` and `workflows.md`. The phasing is a **Recommendation**, chosen to keep the
MVP to "one clinic can run a full OPD day" (`project.md`).

**Phases:** **MVP** = first release · **P2** = Phase 2 · **F** = future expansion.

## 1. Module catalogue

| ID | Module | Phase | Purpose | Depends on | Roles | Main data |
|---|---|---|---|---|---|---|
| M-01 | Platform admin & clinic onboarding | MVP | Super Admin creates, configures, suspends clinics; creates first Clinic Admin | — | Super Admin | clinics, clinic_settings, clinic_members |
| M-02 | Plans & subscriptions | MVP-lite | Plans, subscription periods, expiry, suspension; manual payment recording | M-01 | Super Admin | plans, clinic_subscriptions |
| M-03 | Authentication & sessions | MVP | Login, MFA, password reset, lockout, invites | — | all staff, Super Admin | auth.users, clinic_members, platform_admins |
| M-04 | Staff & permissions | MVP | Invite staff, assign staff type, permission sections/presets, deactivate | M-03 | Clinic Admin | clinic_members |
| M-05 | Clinic settings | MVP | Profile, logo, hours, booking rules, invoice format, notification options | M-01 | Clinic Admin | clinics, clinic_settings |
| M-06 | Departments & services | MVP | Department list; services with price/duration/public flag | M-05 | Clinic Admin | departments, services |
| M-07 | Doctor management | MVP | Doctor profile, NMC reg. no., specialty, fee, public listing | M-04, M-06 | Clinic Admin | doctor_profiles, doctor_departments, doctor_services |
| M-08 | Doctor schedules & availability | MVP | Weekly OPD template, leave/exceptions, slot derivation | M-07 | Clinic Admin, receptionist (view), doctor (own) | doctor_schedules, schedule_exceptions |
| M-09 | Patient registration & profiles | MVP | Register, search, edit demographics; MRN; duplicate warning | M-03 | Receptionist, nurse, doctor, admin | patients |
| M-10 | Appointment management | MVP | Book (desk/phone), confirm, reschedule, cancel, no-show; day/week views | M-08, M-09 | Receptionist, admin, doctor (own) | appointments |
| M-11 | Public clinic website & online booking | MVP | Clinic page, departments, services, doctors, slots, booking, manage-booking link | M-06–M-10 | Patient (anon) | read: public fields; write: patients, appointments |
| M-12 | Reception check-in & queue | MVP | Check-in, token, live queue per doctor, call next, checkout | M-10 | Receptionist, nurse, doctor | visits, token_counters |
| M-13 | Consultation records | MVP | Encounter: vitals, complaint, history, exam, diagnosis, plan, follow-up; finalize; addenda | M-12 | Doctor (write), nurse (vitals) | encounters, vitals, diagnoses, encounter_addenda |
| M-14 | Prescription management | MVP | Rx items, allergy warning, print/PDF with doctor NMC no.; reprint | M-13 | Doctor (issue); reception (print, if permitted) | prescriptions, prescription_items |
| M-15 | Medical history & allergies | MVP-lite | Allergies, chronic conditions, past encounters timeline | M-09, M-13 | Doctor, nurse | patient_allergies, patient_conditions |
| M-16 | Billing & payments | MVP | Invoice per visit from services; discount (permissioned); cash/online/card; receipt; void w/ reason; refunds | M-06, M-12 | Receptionist/cashier, accountant, admin | invoices, invoice_items, payments |
| M-17 | Notifications & reminders | MVP (in-app, push, email) / P2 (SMS) | Staff signals; patient confirmation/reminder/change messages | M-10, M-12 | system; all | notifications, push_subscriptions, notification_outbox |
| M-18 | Reports & analytics | MVP-basic | Appointments/day by doctor, no-show rate, revenue/day, by service, by payment method | M-10, M-16 | Admin, accountant | derived |
| M-19 | Audit log | MVP | Record + view sensitive reads/writes; per-patient access history | M-03 | Admin (view own clinic); Super Admin (platform events only) | audit_log |
| M-20 | Laboratory test management | P2 | Test catalogue, lab orders from encounter, sample status, result entry, report PDF | M-13, M-16 | Doctor (order), lab technician, receptionist (bill) | lab_tests, lab_orders, lab_order_items, lab_results |
| M-21 | Clinical files | P2 | Upload scans/reports (private bucket, signed URLs, audited) | M-13 | Doctor, nurse, lab | clinical_files |
| M-22 | SMS / messaging | P2 | SMS (and possibly Viber/WhatsApp) reminders via outbox | M-17, P-05 | system | notification_outbox |
| M-23 | Patient portal | P2 | Phone-OTP login; my appointments, prescriptions, invoices | M-11, M-14, M-16, P-04 | Patient | patient_accounts (new) |
| M-24 | Online payments | P2 | Booking deposit / invoice payment via Nepali wallets/gateways (eSewa, Khalti, Fonepay: verify) | M-16 | Patient, cashier | payments, payment_intents |
| M-25 | Self-service clinic signup & trial | P2 | Clinic registers, verifies, trial plan; Super Admin approves | M-01, M-02 | Prospect, Super Admin | clinics (pending) |
| M-26 | Follow-ups & referrals | P2 | Follow-up booking from encounter; internal referral grants access | M-10, M-13 | Doctor | appointments, care_access_grants |
| M-27 | Daily summary email / PDF | P2 | Owner's end-of-day report (reuse HRestroSewa reporting service shape) | M-16, M-18 | Admin | report_deliveries |
| M-28 | Patient merge | P2 | Merge duplicates, audited and reversible | M-09 | Admin | patients, merge_log |
| M-29 | Pharmacy & medicine | F | Medicine master, stock (derived, HRestroSewa pattern), dispensing against Rx, batches/expiry | M-14, M-16 | Pharmacist | medicines, stock movements, dispensations |
| M-30 | Insurance (HIB/SSF) | F | Claims for Nepal Health Insurance Board / Social Security Fund (verify eligibility rules) | M-16 | Accountant | claims |
| M-31 | Multi-branch | F | Branches under one clinic org | all | Admin | branch_id |
| M-32 | Telemedicine | F | Video consult + e-prescription | M-13, M-14 | Doctor, patient | — |
| M-33 | Interoperability | F | FHIR export, lab device / LIS integration | M-13, M-20 | — | — |
| M-34 | Custom domains & advanced site builder | F | Clinic's own domain, richer site content | M-11 | Admin | clinic_domains |

## 2. Feature inventory by surface

### Super Admin
- Clinic list (status, plan, expiry, doctors/staff counts, last activity).
- Create clinic (M-01), edit, suspend/reactivate, close.
- Create or reset the first Clinic Admin invite.
- Plans CRUD, record subscription payments and extensions, expiry warnings (M-02).
- Platform usage: clinics active, appointments/day across the platform (counts only), online
  booking volume, notification delivery failures, error rates.
- Platform settings: sender identity, SMS provider config (P2), feature flags per plan.
- Platform audit view: platform-admin actions only. **No patient records, no clinical
  views, and no clinic data beyond aggregate counts** (D-08).

### Clinic Admin
- Dashboard: today's appointments, queue status, revenue, no-shows.
- Staff: invite, staff type, permission preset or custom, deactivate, reset access (M-04).
- Doctors, departments, services, fees, schedules, leave (M-06–M-08).
- Clinic settings: profile, logo, public site toggle, booking rules, invoice format,
  notification options (M-05).
- Appointments, billing and reports (all operational modules).
- Audit log viewer for the clinic: who accessed which patient and when (M-19).
- Clinical content only if the admin is also a doctor, or through break-glass (D-06).

### Staff (by staff type; defaults are presets, see `permissions.md`)
- **Receptionist:** patient register/search/edit demographics, appointments (all doctors),
  check-in, queue board, billing and receipts, print prescriptions (if permitted).
- **Doctor:** my schedule, my queue, consultation screen, prescriptions, patient history
  within the care relationship, my leave requests (or admin-set), follow-ups.
- **Nurse:** queue, vitals entry for checked-in patients, view allergies.
- **Lab technician (P2):** lab worklist and result entry.
- **Pharmacist (F):** dispensing.
- **Accountant:** billing, payments, reports. No clinical access.

### Patient website (public)
- Clinic home: name, logo, address with map link, phone, OPD hours, departments, services
  and fees (if the clinic chooses to show them).
- Doctor list and profiles: photo, qualification, specialty, NMC no. (displayed),
  OPD days/times.
- Availability: choose department or doctor, then date, then slot (or token availability).
- Booking: name, phone, age/DOB, sex, optional email and reason, plus consent tick. The
  result is a confirmation screen with a reference number and the manage link.
- Manage booking: view, cancel (before the cut-off) and reschedule (into an open slot).
- Messages: confirmation and reminders by email (MVP) and SMS (P2).
- Bilingual content (English/Nepali) P2, i18n-ready in the MVP (P-13).

## 3. MVP summary
M-01 · M-02 (lite) · M-03 · M-04 · M-05 · M-06 · M-07 · M-08 · M-09 · M-10 · M-11 · M-12 ·
M-13 · M-14 · M-15 (lite) · M-16 · M-17 (in-app/push/email) · M-18 (basic) · M-19.

**Deliberately not in the MVP, with reasons:** lab (M-20), because it needs its own
workflow and result model; pharmacy (M-29), because stock, batches and the drug regulation
questions are large; SMS (M-22), because provider and cost are undecided (P-05) and email
plus a printed slip covers the gap; patient portal (M-23), because the manage-booking link
covers the MVP need without patient accounts; online payment (M-24), because it needs a
gateway contract; self-signup (M-25), because hand onboarding by the Super Admin is safer
for the first clinics.
