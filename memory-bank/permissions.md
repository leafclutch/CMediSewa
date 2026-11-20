# Permissions

Status: **Recommendation.** The structure is adapted from HRestroSewa's `lib/permissions.ts`
(single source of truth, sections → levels → extras, presets, navigation derived from
permissions, server re-checks everywhere). The medical additions are **staff-type gating**,
**care-relationship record access**, **break-glass** and the rule that the **Clinic Admin
does not bypass clinical content**.

## 1. Three questions, three mechanisms
Every request that touches data is checked on three independent axes:
1. **Which clinic?** Tenant. Resolved from the live `clinic_members` row and never taken from
   the client (`architecture.md` §4).
2. **May this person do this kind of thing?** Capability. `permissions text[]` on the
   membership, checked by `hasPermission(ctx, key)`.
3. **May this person do it to this patient?** Record level. The care relationship,
   checked by `lib/care-access.ts` and in SQL inside the clinical RPCs.

Passing (2) never widens (3). HRestroSewa applies the same rule with `close_bills`, which
does not widen which tables a waiter sees.

## 2. Roles
| Role | Stored as | Meaning |
|---|---|---|
| Super Admin | `platform_admins` | Platform operator, outside every clinic. Not a clinic member. |
| Clinic Admin | `clinic_members.role = 'clinic_admin'` | Holds **all operational permissions implicitly**. **Clinical permissions are not implied** (D-06). |
| Clinic Staff | `clinic_members.role = 'clinic_staff'` | Holds exactly `permissions[]`. |

**Staff type** (`staff_type`: doctor, nurse, receptionist, lab_technician, pharmacist,
accountant, other) is *not* a permission. It does two things:
- picks the default **preset**, and
- **limits which clinical keys can be granted.** Only `doctor` can hold
  `write_consultation` / `issue_prescriptions`. Only `doctor`, `nurse` and `lab_technician`
  can hold `view_clinical`. Every other type is refused by the staff-update action and by a
  DB CHECK/trigger. This follows HRestroSewa's rule that a module absent for a business type
  can't be granted.

A Clinic Admin who is also a doctor has `staff_type = doctor` and receives the doctor's
clinical keys through the same grant path as any doctor.

## 3. Permission keys (proposed)
Write implies read within a section.

| Section | Keys | Notes |
|---|---|---|
| Patients | `view_patients`, `manage_patients` | Demographics only (name, contact, MRN, address). Not clinical. |
| Appointments | `view_appointments`, `manage_appointments` | Clinic-wide. A doctor without these still sees **own** appointments implicitly. |
| Schedules | `view_schedules`, `manage_schedules` | `manage_own_leave` extra for doctors. |
| Queue / reception | `view_queue`, `manage_queue` | Check-in, token, call next, checkout. |
| Clinical | `record_vitals`, `view_clinical`, `write_consultation`, `issue_prescriptions` | All care-relationship scoped (§5). Type-gated (§2). |
| Clinical extras | `print_prescriptions`, `view_all_clinical`, `break_glass` | `print_prescriptions` lets reception reprint an issued Rx without seeing the chart. `view_all_clinical` is a rare, audited clinic-wide read (e.g. a medical director). `break_glass` is emergency access (§6). |
| Billing | `view_billing`, `manage_billing` | Create and issue invoices, take payments. |
| Billing extras | `apply_discounts`, `void_invoices`, `issue_refunds` | Separate trust levels, as in HRestroSewa's split of purchases and vendors. |
| Reports | `view_reports`, `view_financial_reports` | Operational counts vs money. |
| Lab (P2) | `view_lab`, `manage_lab_orders`, `enter_lab_results` | Results are clinical, so they are care-scoped. |

**Admin-only (not delegable in the MVP):** staff and permission management, clinic
settings, doctor and service setup, audit log viewing, subscription view. *Reason:* fewer
keys means fewer mistakes, and HRestroSewa removed its delegable settings keys for the same
reason. They can become delegable later if real clinics ask for it.

**Rule copied from HRestroSewa:** never add a key that gates nothing. Remove dead keys with
a migration that rewrites stored arrays.

## 4. Presets (defaults per staff type)
| Preset | Keys |
|---|---|
| Receptionist | view/manage_patients, view/manage_appointments, view_schedules, view/manage_queue, view/manage_billing, print_prescriptions |
| Doctor | view_patients, view_schedules, manage_own_leave, view_queue, record_vitals, view_clinical, write_consultation, issue_prescriptions, print_prescriptions |
| Nurse | view_patients, view_queue, manage_queue, record_vitals, view_clinical (scope-limited, §5) |
| Accountant | view_billing, manage_billing, view_reports, view_financial_reports |
| Lab technician (P2) | view_patients, view_queue, view_lab, enter_lab_results |
The editor shows one control per section (None / View / Full + extras), and a non-preset
combination shows as "Custom", exactly as in HRestroSewa.

## 5. Record-level access: the care relationship
`canAccessClinical(member, patient, purpose)` returns `true` if **any** of these holds. The
same function (in TS) and the same SQL predicate are used by pages, actions, RPCs, push
recipient resolution and exports.

| # | Rule | Applies to | Scope granted |
|---|---|---|---|
| C1 | Member is the **doctor** on an appointment/visit for P that is today or upcoming (non-cancelled) | doctors | full chart read + write for that encounter |
| C2 | Member is the doctor of **any past encounter** with P | doctors | full chart read (history). Write only to their own encounters/addenda |
| C3 | P has an **open visit today** and member holds `record_vitals` | nurses | vitals, allergies, current-visit summary only. **No** past notes |
| C4 | An **explicit grant** in `care_access_grants` (referral or share), not expired | anyone with `view_clinical` | as granted, time-boxed |
| C5 | Member holds `view_all_clinical` | rare | full read, **every read audited** with reason optional |
| C6 | **Break-glass** session is open for (member, P) | holders of `break_glass` | full read for 60 min (configurable), reason mandatory |

**[OPEN] P-07:** C2 lets a doctor keep reading a patient's history indefinitely after one
visit. The alternative is to limit it to N months. The recommended default is C2 as written,
because doctors need longitudinal history, every read is audited, and the clinic can revoke
by deactivating the member.

**What is clinical:** encounters, vitals, diagnoses, notes, addenda, prescriptions (content),
allergies, conditions, lab results and clinical files. **Not clinical:** demographics,
appointment times, the doctor booked, the visit/token, invoices. Invoice lines name
services, which can imply a condition, so `view_billing` is a trusted permission
(**Assumption**: acceptable for small clinics).

**Reason for visit** typed by the patient at booking is treated as clinical-adjacent. Reception
sees it, because triage needs it. It is never sent in push, SMS or email.

## 6. Break-glass
- Who: holders of `break_glass` (by default nobody; admin grants it to senior clinicians).
- How: the member chooses "Emergency access", enters a reason (required, at least 10 chars)
  and re-authenticates. The RPC opens a `care_access_grants` row of kind `break_glass`
  that expires after 60 minutes, writes an `audit_log` entry, and creates an in-app
  notification for every Clinic Admin.
- Review: the admin audit view lists break-glass events that haven't been reviewed.

## 7. Receptionist access (explicit)
**Can:** search and register patients, edit demographics, book, reschedule and cancel for
any doctor, check in, manage the queue, bill and take payments, and reprint an issued
prescription when they hold `print_prescriptions` (the PDF render is audited as
`prescription.print`).
**Cannot:** open encounters, notes, diagnoses, history or allergies (beyond an "allergy
alert exists" flag if P-07 adopts it), see lab results, or change prices without
`apply_discounts`.

## 8. Clinic Admin access (explicit)
All operational modules, setup, staff, settings, reports and the audit log. **No implicit
clinical access** (D-06). If the admin is a doctor, the doctor rules apply. Otherwise the
admin uses break-glass, which is audited and visible to the other admins. A single-admin
clinic will see its own break-glass in its own log, and that log can't be edited.
**[OPEN] P-07:** some owner-doctors may find this restrictive. The alternative is a clinic
setting "Admin has clinic-wide clinical read (audited)", off by default.

## 9. Super Admin access (explicit)
Clinics, plans, subscriptions, platform settings, platform metrics (aggregate counts) and
platform-admin audit. **No route, action or RPC exposes patient rows or clinical data to
platform admins** (D-08). Support that needs clinic data is done by the clinic admin sharing
a screen, or later through an "audited support session" the clinic must approve (future).
Super Admins can suspend a clinic or reset its admin's access, and both actions are audited.

## 10. Inheritance & exceptions
- **Write implies read** within a section (`manage_x` ⇒ `view_x`).
- **Clinic Admin ⇒ all operational keys.** It does **not** imply clinical keys.
- **Doctor ⇒ own appointments and own schedule**, even without the clinic-wide keys.
- **Staff-type gating overrides any stored permission:** a clinical key on a
  non-clinical type is inert and refused on write.
- **A deactivated member or a suspended clinic has no access**, checked live on every
  request.
- **Exports** (CSV/PDF of patient lists or clinical data) need the underlying permission
  **plus** a per-export audit entry. Clinical exports are admin-only in the MVP.
- **Navigation is derived from permissions**, so it can't drift from what the guards
  allow. Client-side gating is convenience only; every page and every action re-checks on
  the server.
