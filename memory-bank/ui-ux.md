# UI / UX

Status: **Recommendation.** Visual design is not final. A dedicated design pass (tokens,
type, components) happens in Phase 0 and is reviewed by the owner before screens are built.

## 1. Design direction
- **Calm, clinical, trustworthy.** A light neutral canvas, one confident primary colour
  (a medical teal or blue; **[OPEN]**, decided with the brand), and semantic colours used only
  for meaning: red = allergy/critical, amber = waiting/late, green = done/paid.
- **Speed at the desk over decoration.** Reception works with a queue in front of it.
  Every frequent action is one tap from the dashboard, search is always reachable
  (`/` shortcut on desktop) and the keyboard works on the PC.
- **Readable for everyone:** 16px base, large touch targets (≥44px), high contrast
  (WCAG AA), tabular numerals for money, times and tokens.
- **Dark mode** for staff (night OPD), driven by tokens, reusing HRestroSewa's theme toggle
  and theme-sync approach. Its CSS traps (`@theme inline`, animation `fill-mode: both`) are
  also carried over so they aren't reintroduced.
- **Bilingual-ready:** layouts must tolerate Devanagari strings that run about 30% longer.
  Nepali glyphs need a fallback font (Noto Sans Devanagari, for example).
- **Dates:** show AD with BS alongside wherever patients or printed documents see a date
  (e.g. "2026-11-20 · २०८३ मंसिर ४"). Format is a clinic setting.

## 2. Dashboard structure (clinic)
```
┌────────────┬───────────────────────────────────────────────────────────┐
│  Logo       │  Top bar: clinic name · global patient search · bell ·   │
│             │           user menu (profile, theme, logout)             │
│  Today      ├───────────────────────────────────────────────────────────┤
│  Queue      │  Page content                                              │
│  Appts      │   e.g. Today: KPI tiles (booked / checked-in / waiting /   │
│  Patients   │   completed / no-show / revenue) + per-doctor queue cards  │
│  Consult*   │                                                            │
│  Billing    │                                                            │
│  Reports    │                                                            │
│  ───────    │                                                            │
│  Setup ▸    │  (Doctors, Departments, Services, Schedules, Staff,        │
│  Audit      │   Settings: admin-only)                                    │
└────────────┴───────────────────────────────────────────────────────────┘
 * "My queue / Consultations" appears for doctors.
```
- **The sidebar is derived from permissions** (`getClinicNav(ctx)`), so a receptionist sees
  Today, Queue, Appointments, Patients and Billing, and a doctor sees Today, My Queue,
  My Appointments and Patients (care-scoped).
- **Landing page per staff type:** doctor → My Queue, receptionist → Today,
  accountant → Billing, admin → Today.

## 3. Responsive strategy
- **Reception PC (≥1280px):** sidebar plus a two-pane layout (queue list | detail).
- **Tablet (768–1279px):** collapsible sidebar. The consultation screen uses tabs
  (Summary / Notes / Rx).
- **Phone (<768px):** a bottom tab bar (four items plus "More") replaces the sidebar, forms
  are single-column, and tables become cards. Doctors on phones mainly need My Queue,
  the patient summary and a quick Rx.
- No horizontal scrolling on any page. Data tables collapse to cards on small screens.

## 4. Super Admin interface
A plain, dense admin UI with no clinic branding: a Clinics table (status, plan, expiry,
counts), a clinic detail page (profile, subscription history, admin invite, suspend), a Plans
page, platform Metrics and Settings. The visual language stays deliberately separate from the
clinic UI so a platform admin always knows which context they are in. A red banner appears
on any destructive action.

## 5. Clinic Admin interface
The clinic dashboard plus **Setup** (guided at first run, see the onboarding checklist in
W-02): Clinic profile → Departments → Services and fees → Doctors → Schedules → Staff →
Booking rules → Publish website. Each step shows a completion state. The **Audit** page has
filters by patient, staff member, action and date, plus a "break-glass to review" list.

## 6. Staff dashboards
- **Reception, Today:** the queue board per doctor (token, patient, status chip, wait time),
  a "Check in" button on each upcoming appointment, a "New walk-in" action, and a quick
  register form.
- **Queue board display** (optional TV/monitor view, P2): token numbers and doctor room
  only, **no patient names**, read-only, live over SSE.
- **Doctor, My Queue:** ordered list, then "Start consultation".
- **Consultation screen:** a patient header (name, age/sex, MRN, **allergy banner in red**),
  a vitals strip, and sections for Complaint, History, Exam, Diagnosis, Plan and Follow-up,
  plus an Rx builder (medicine, dose, frequency, duration, with quick-repeat from last Rx)
  and a "Finalize and print" button. Autosave runs on drafts. A finalized record is
  read-only and has "Add addendum".
- **Billing:** visit → services pre-filled from the appointment → discount (if permitted)
  → payment method(s) → "Issue and print". Thermal 80mm and A4/A5 receipt layouts, reusing
  HRestroSewa's print-width setting idea.

## 7. Patient-facing website
- A mobile-first, light, fast page (most traffic is phones on mobile data). No client-heavy
  bundles: Server Components, minimal JS and optimised images.
- Clinic branding: logo, primary colour from the clinic, name and address.
- Sections: hero (name, hours, call button, "Book appointment"), departments, doctors
  (cards with photo, specialty and OPD days), services and fees (optional), location
  (map link), contact.
- SEO: per-clinic metadata, `MedicalClinic` / `Physician` schema.org JSON-LD and a sitemap
  entry for each published clinic. HRestroSewa's SEO landing work is a reference for this.

## 8. Appointment booking experience
```
1 Choose  ──►  2 Pick time  ──►  3 Your details  ──►  4 Confirm  ──►  ✓ Booked
 department     date strip         name, phone,         summary +        ref no. +
 or doctor      + slot chips       age/DOB, sex,        consent tick     manage link
                (or "token for     email (opt),         + CAPTCHA        (+ SMS/email
                 this OPD")        reason (opt)                          when enabled)
```
- At most 4 steps. A step indicator is shown and the back button never loses entered data.
- Slots show only real availability (derived server-side). Taken slots disappear rather
  than erroring later. If a race does happen, the screen says "That time was just taken,
  here are the nearest open times" and keeps the form.
- Phone is the required contact (+977 format with validation). Email is optional.
- The confirmation screen is printable and screenshot-friendly: clinic, doctor,
  date/time (AD + BS), token if any, reference number, arrival instructions and the
  manage link.
- Cancel and reschedule from the manage link, with the clinic's cut-off rule shown upfront.
- Accessibility: labels on every field, an error summary, and nothing conveyed by colour
  alone.

## 9. Reusable design components
Copied from HRestroSewa `components/ui` and restyled: Button, Input, Skeleton, SearchSelect,
PeriodFilter, CountPill, ThemeToggle/ThemeSync, PaymentMethodPicker, plus the PWA set
(InstallPrompt, OfflineGate, PushPrompt, RegisterSW, PullToRefresh).
**New for CMediSewa:** PatientHeader (with AllergyBanner), StatusChip (appointment state
machine colours), SlotPicker, DateStrip (AD + BS), TokenBadge, VitalsStrip, RxItemRow,
KPI Tile, DataTable→Cards responsive wrapper, AuditTrail list, ConfirmDialog (with reason
input for voids, cancellations and break-glass), PrintLayout (Rx, receipt).
