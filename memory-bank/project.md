# Project: CMediSewa

> **How to use this memory bank.** Read `project.md` → `architecture.md` → `decisions.md` →
> `current-task.md` first. Load `database.md`, `permissions.md` and `security.md` for anything
> that touches data, access or patient information. Edit only the affected files when work
> finishes. `decisions.md` is append-only. Mark claims **Confirmed** / **Assumption** /
> **Recommendation**. Cross-reference IDs: `D-xx` = decision, `P-xx` = pending decision,
> `M-xx` = module (`features.md`), `W-xx` = workflow (`workflows.md`).

## Overview
CMediSewa is a multi-tenant SaaS platform for small medical clinics and healthcare centres. One
platform hosts many clinics. Each clinic gets a management dashboard and a public website, and
a Super Admin layer operates the platform. Its architectural reference is HRestroSewa (see
`../docs/hrestrosewa-reference.md`), but CMediSewa is an independent product with its own
database, deployment and business logic.

## Product vision
A small clinic in Nepal should be able to run its day on CMediSewa from a phone, tablet or
reception PC: patients book online or at the desk, get a token, see the doctor, leave with a
printed prescription and receipt, and the clinic owner sees the day's numbers. Patient records
must be safer than paper, not less safe: access is limited by care relationship, every read of
a clinical record is audited, and nothing is lost.

## Target users
- **Clinic owners/managers** (Clinic Admin): often a doctor-owner, sometimes a non-clinical manager.
- **Doctors**: general practitioners and specialists. Many are visiting doctors who keep OPD
  hours at several clinics (**Assumption**, common in Nepal; it drives schedule design).
- **Nurses / health assistants**, **receptionists**, **lab technicians**, **pharmacists**,
  **accountants**.
- **Patients**: book through a phone browser. Many have no email address, so a phone
  number is the primary contact (**Assumption**).
- **Platform operator** (Super Admin).

## Target market
Small private clinics, polyclinics, specialist OPD clinics, dental and eye clinics, and small
diagnostic and healthcare centres in Nepal. Typical size is 1–15 doctors and 2–30 staff, in a
single location. **Out of scope for the MVP:** hospitals with inpatient wards, operation
theatres, multi-branch chains and insurance billing.

## Main objectives
1. Cut front-desk friction: online booking, token queue, fast patient lookup.
2. Give doctors a quick consultation screen and clean printable prescriptions.
3. Keep billing simple, correct and sequential.
4. Protect patient data from the start with tenant isolation, care-relationship access,
   audit logging and backups.
5. Make onboarding a new clinic a configuration task, not a code change.

## Scope
**In scope (product):** platform administration, clinic administration, staff and permissions,
doctors and schedules, patients, appointments and online booking, reception queue,
consultations, prescriptions, billing, notifications, reports, public clinic website.
Lab, pharmacy, patient portal and online payments come in later phases (`features.md`).

**Out of scope (for now):** inpatient/IPD, OT scheduling, insurance claims (HIB/SSF),
telemedicine video, LIS/PACS device integration, native mobile apps.

## Core application structure
| Surface | Route group (proposed) | Users |
|---|---|---|
| Marketing site | `(marketing)/` | prospects |
| Super Admin | `(superadmin)/superadmin/*` | platform operator |
| Clinic dashboard | `(clinic)/clinic/*` | Clinic Admin + all staff; navigation derived from permissions |
| Public clinic website | `(public)/c/[slug]/*` | patients, anonymous |
| Auth | `(auth)/login`, `/superadmin/login` | everyone signing in |

HRestroSewa has separate `(admin)` and `(employee)` trees. CMediSewa uses **one** clinic
dashboard whose navigation and pages are gated by permission (**Recommendation**, D-07).
A clinic's admin and its staff work in the same screens with different access, which avoids
building every screen twice.

## MVP definition
The MVP is done when one real clinic can run a full OPD day:
1. The Super Admin onboards the clinic and creates its admin.
2. The admin sets up departments, services and fees, doctors and their weekly schedules, and
   staff with roles.
3. A patient books online, or reception books by phone. The booking is confirmed, and double
   booking is impossible.
4. Reception checks the patient in and a token joins the live queue.
5. The nurse records vitals (optional) and the doctor records the consultation and issues a
   prescription, which prints.
6. Reception bills the visit, takes payment (cash/online) and prints a receipt with a
   sequential invoice number.
7. The admin sees daily appointments, no-shows and revenue.
8. Every clinical read and write is audited, backups are running and verified, and no
   staff member can see a record outside their care relationship.

The MVP module list is `features.md` → "MVP". The build order is `roadmap.md`.

## Glossary
- **MRN**: medical record number, unique per clinic.
- **Encounter / consultation**: one doctor–patient interaction tied to a visit.
- **Visit**: a patient's presence at the clinic on a day (check-in to checkout). It may hold
  an appointment, an encounter and an invoice.
- **Token**: the daily queue number given at check-in.
- **Care relationship**: the link between a staff member and a patient that grants clinical
  access (`permissions.md`).
- **PHI**: protected health information, meaning any data that identifies a patient together with health data.
- **BS / AD**: Bikram Sambat (Nepali calendar) / Gregorian dates.
