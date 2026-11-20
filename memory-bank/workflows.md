# Workflows

End-to-end flows the data model and permissions must support. Status: **Recommendation.**
Each step names the actor, the system action and the important rule. `RPC:` marks an
atomic database function. Cross-refs: modules (M-xx), decisions (D-xx), pending (P-xx).

## W-01 Clinic registration & onboarding (M-01, M-02)
1. **Super Admin** opens Clinics → New and enters the name, slug (`[a-z0-9-]`, unique),
   contact details, plan and subscription start.
2. The system creates the `clinics` row (status `active`, `public_booking_enabled = false`),
   default `clinic_settings` and the first `clinic_subscriptions` row.
3. The Super Admin enters the owner's email. The system creates a `clinic_members` row
   (`clinic_admin`) and sends an **invite link** (single-use, expiring). The owner sets a
   password and enrols MFA (P-11).
4. Audit: `clinic.create`, `member.invite`.
*P2 (M-25):* a self-service signup form creates a `pending` clinic for Super Admin approval.

## W-02 Clinic setup (M-05–M-08)
A guided checklist for the Clinic Admin. The public website can't be published until the
required steps are done.
1. Profile: legal name, PAN/registration no., address, phone, logo (public bucket),
   OPD hours.
2. Departments → Services (price, duration, public flag).
3. Doctors (W-03), then staff: invite by email, pick the staff type (which gives a preset),
   adjust permissions.
4. Booking rules: booking window, minimum notice, cancellation cut-off, scheduling mode
   (P-03), and whether service prices show publicly.
5. Invoice settings: prefix, numbering pad, receipt footer and paper width.
6. **Publish website**, which sets `public_booking_enabled`. The system refuses if no doctor
   has a schedule.

## W-03 Doctor registration & availability setup (M-07, M-08)
1. The Admin invites the doctor (staff type `doctor`) → doctor preset. The doctor accepts
   and sets a password and MFA.
2. Admin or doctor fills in the profile: NMC registration no. (required, unique per clinic;
   **[OPEN]** verification against the NMC register is manual), qualification, specialty,
   departments, services offered, default fee, photo and `is_public`.
3. Weekly schedule template: for example Sun/Tue/Thu 16:00–19:00, 15-minute slots, or a
   token mode with a maximum of 30. The exclusion constraint rejects overlapping windows.
4. Exceptions: leave on a date → slots disappear. If confirmed appointments already exist in
   that range, the system lists them for reception to reschedule (W-11) and does **not**
   auto-cancel.
5. Effective dating: a template change applies from `effective_from`, so booked future
   appointments are kept.

## W-04 Patient registration (M-09)
**At the desk:**
1. Reception searches by phone, name or MRN **before** creating anyone.
2. Candidate duplicates (same phone + similar name/age) are shown. Reception picks an existing
   patient or confirms "new".
3. The system creates the patient, and a trigger stamps the MRN (`<prefix>-<BS year>-<seq>`,
   **Recommendation**). Consent for data processing and messaging is recorded (P-10).
**Online:** the booking (W-05) creates or matches the patient by `(clinic, phone, name,
age/DOB)`. A fuzzy match is **not** auto-merged. It becomes a `created_via = online` record
flagged "possible duplicate" for reception to resolve at check-in.

## W-05 Online appointment booking (M-11, M-10)
1. The patient opens `/c/<slug>` and chooses a department or doctor, then a date.
2. The server returns availability from `available_slots(...)`: the template, minus
   exceptions, minus active appointments, minus anything inside the minimum-notice window
   or past the booking window.
3. The patient picks a slot (or "token for this OPD") and enters name, phone, age/DOB,
   sex, optional email, optional reason and the consent tick. A CAPTCHA check runs.
   OTP verification runs if P-04 adopts it.
4. **RPC `book_appointment`** (one transaction):
   - checks that the clinic is active, booking is enabled, the doctor is public and active,
     and the slot is in a valid window;
   - enforces the rate and abuse limits (future bookings per phone, per IP, per day);
   - matches or creates the patient (W-04, online rules);
   - inserts the appointment with status `confirmed` (or `requested` if the clinic requires
     manual approval, a clinic setting). **The GiST exclusion constraint makes the race
     safe:** the second booker gets `SLOT_TAKEN`;
   - generates the manage token (only its hash is stored) and writes `notification_outbox`
     rows (confirmation, and a reminder with `send_after = start − 24h`);
   - writes `audit_log` `appointment.create` (actor `patient_token`).
5. The patient sees the confirmation (reference no., manage link, AD + BS date).
6. Staff get an SSE `appointments` signal and an in-app notification, plus push to that
   doctor and reception if they have opted in, with a PHI-minimal payload (D-10).

## W-06 Appointment confirmation
- **Auto-confirm (default):** the booking is `confirmed` immediately and the confirmation
  is sent at once by the outbox worker.
- **Manual approval (clinic setting):** the booking is `requested` and reception confirms
  or declines it from the Appointments list. The patient is then notified. Declined
  bookings release the slot.
- **Phone or desk bookings** (M-10) are made by reception through the same RPC
  (`source = phone`), with the same conflict rules. A confirmation goes to the patient if a
  contact exists.

## W-07 Reception check-in (M-12)
1. The patient arrives. Reception finds the appointment (today's list or the reference no.)
   or creates a walk-in (patient search/register → appointment `source = walk_in`, no time
   range in token mode).
2. Reception resolves any duplicate flag and verifies or updates contact details.
3. **RPC `check_in`**: creates the `visit`, assigns `token_no` from `token_counters` with a
   row lock (no duplicate tokens), and moves the appointment to `checked_in`. It refuses
   non-today appointments and those already checked in.
4. The queue board updates over SSE. The token slip prints (token, doctor, room).
   *P2:* the patient is also sent an SMS.
5. Optional triage: the nurse records **vitals** on the open visit (care rule C3).
6. A no-show is marked automatically by cron at OPD end + grace period, or manually.

## W-08 Doctor consultation (M-13, M-15)
1. The doctor opens My Queue and taps "Call next" or picks a token. This sets
   `in_consultation`, and the queue display shows the token.
2. "Start consultation" → **RPC `open_encounter`** checks care rule C1, creates a draft
   `encounter` linked to the visit, and audits `encounter.create`.
3. The chart loads through **RPC `get_patient_chart`**. It checks the care rule and writes
   `audit_log` `patient.chart_read` in the same transaction. The chart shows demographics,
   the **allergy banner**, active conditions, vitals and past encounters (C2 history).
4. The doctor writes the complaint, history, exam, diagnosis (free text, optional ICD-10 per
   P-15), plan and follow-up date. Drafts autosave.
5. **Finalize** → **RPC `finalize_encounter`**: sets `finalized_at`. From then on a trigger
   blocks edits and corrections are addenda only. Audit `encounter.finalize`.
6. The appointment moves to `completed`, or stays "awaiting billing" until checkout.

## W-09 Prescription generation (M-14)
1. In the consultation, the doctor adds Rx items: medicine (free text with suggestions from
   the doctor's own recent prescriptions), strength, form, dose, frequency, route, duration,
   quantity and instructions.
2. **Allergy check:** if an item's text matches a recorded allergen (simple matching in the
   MVP), the doctor sees a blocking warning and must confirm with a reason.
3. **RPC `issue_prescription`**: stamps `rx_number`, sets `issued`, freezes the record and
   audits `prescription.issue`.
4. Print or PDF (pdf-lib): clinic header, doctor name, qualification and **NMC reg. no.**,
   patient name, age/sex and MRN, date (AD + BS), diagnosis (doctor's choice to print),
   items, advice, follow-up and a signature line. **[OPEN]** Whether a digital signature or
   stamp is needed, and what printed-Rx content regulations require (P-08 sibling; verify
   with the NMC / Department of Drug Administration rules).
5. A reprint by reception needs `print_prescriptions` and is audited as `prescription.print`.
6. A mistake means **cancel and reissue**. The cancel needs a reason and is audited. The old
   Rx stays visible as cancelled.

## W-10 Billing & payment (M-16)
1. At checkout, reception opens the visit. **RPC `create_invoice_for_visit`** pre-fills
   lines from the appointment's service or doctor fee (price **snapshotted**) and any
   procedures added.
2. Reception adds or removes lines and applies a discount if it holds `apply_discounts`.
   The discount reason is recorded and audited.
3. **RPC `issue_invoice`**: stamps `invoice_number` (sequential per clinic per Nepali fiscal
   year, never reused, with gaps impossible except for voids, which keep their number) and
   freezes the lines.
4. **RPC `record_payment`**: cash, online (wallet/QR reference) or card, split payments
   allowed, with the amount at most the balance. Balance = total − payments (derived).
5. Print the receipt (thermal 80mm or A4/A5). The visit is checked out.
6. **Void** (needs `void_invoices`, a reason and is audited) keeps the row and its number,
   marked void. **Refund** (needs `issue_refunds`) is a negative payment with a reason.
7. **[OPEN] P-08:** VAT and IRD rules may add mandatory invoice fields, a format, or a
   requirement to register the billing software. Credit/due billing is Phase 2.

## W-11 Appointment cancellation & rescheduling (M-10, M-11)
- **By the patient** (manage link): cancellation is allowed until the cut-off (clinic
  setting). Reschedule = **RPC `reschedule_appointment`**, which in one transaction checks
  the new slot (exclusion constraint), creates the new appointment with
  `rescheduled_from_id` and marks the old one `cancelled` (reason "rescheduled"). The outbox
  sends a notice, and old reminders are cancelled.
- **By staff:** the same RPCs with `cancelled_by_kind = staff` and a mandatory reason.
- **By doctor leave** (W-03 §4): the affected appointments are listed, and reception
  contacts patients and reschedules or cancels in bulk (the bulk action sends notices).
- Late cancellation and no-show counts per phone are recorded. They feed the online-booking
  abuse limits (W-05).

## W-12 Notifications & reminders (M-17)
- **Patient messages** (`notification_outbox`): booking confirmed, reminder (24h before,
  configurable; a same-day morning reminder is optional), rescheduled, cancelled, and
  doctor on leave. The cron route `/api/cron/dispatch-notifications` runs every 5 minutes
  (pg_cron → secret-gated route, D-10). It sends rows where `send_after ≤ now()` and the
  status is pending, with retries and backoff. `unique(appointment_id, kind, channel)`
  guarantees each message is sent once. Cancelling an appointment cancels its pending
  reminders.
- **Content rule:** clinic name, doctor name, date/time, token, manage link and clinic phone.
  **Never** a diagnosis, reason for visit or prescription.
- **Channels:** email in the MVP (Resend), SMS in P2 (P-05). The outbox routes by
  `channel`, so SMS is a new sender, not a redesign.
- **Staff signals:** in-app `notifications` plus web push, with recipients resolved through
  the visibility predicate (only staff who could see the event). Push payloads use tokens
  and doctor names. Patient names appear only if the clinic enables it.
- **Platform crons:** subscription expiry (daily, deactivates lapsed clinics, following
  HRestroSewa's `subscription-expiry`), automatic no-show marking, and retention jobs (P-10).
