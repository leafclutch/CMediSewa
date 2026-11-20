# CMediSewa

**CMediSewa** is a planned multi-tenant SaaS platform for **small medical clinics and
healthcare centres in Nepal**. One deployment serves many clinics. Each clinic gets:

- an internal **management dashboard** for its admin, doctors, nurses, receptionists and
  other staff, and
- a separate public **clinic website** where patients can see the clinic's services and
  doctors, check open appointment slots and book online.

A **Super Admin** layer runs the platform itself: onboarding clinics, plans and
subscriptions, platform settings and usage monitoring.

> **Status: architecture & planning. No application code yet.**
> The documentation in `memory-bank/` has to be reviewed and the pending decisions in
> `memory-bank/decisions.md` resolved before Phase 0 (scaffold) starts. See
> `memory-bank/current-task.md`.

## Who it is for

| User | What they do |
|---|---|
| Platform operator (Super Admin) | Registers clinics, manages plans/subscriptions, monitors the platform. **Never sees patient data.** |
| Clinic Admin (owner / manager) | Runs one clinic: staff, doctors, departments, services, schedules, settings, billing, reports. |
| Clinic staff | Receptionist, doctor, nurse, lab technician, pharmacist, accountant. Each sees only the modules **and the patient records** their role and care relationship allow. |
| Patients | Book, reschedule and cancel appointments on the clinic's public site. No account in the MVP. |

## Architecture at a glance

```
                 ┌──────────────────────── one Next.js app ───────────────────────┐
 Browser  ──►    │ (marketing)  (public)/c/[slug]  (auth)  (clinic)  (superadmin)  │
                 │      │             │               │         │          │       │
                 │      └──── Server Components + Server Actions (the only API) ─┘ │
                 │                    │ service_role, server-only                  │
                 └────────────────────┼───────────────────────────────────────────┘
                                      ▼
            Supabase (self-hosted): Postgres + RLS deny-all backstop + RPCs,
            Auth (email+password, TOTP MFA), Storage (public: logos; private: clinical)
                                      │
            pg_cron → secret-gated cron routes (reminders, subscription expiry)
            LISTEN/NOTIFY → SSE realtime bus (topic names only, never row data)
            web-push (VAPID) · Resend email (HTTPS) · SMS provider (pending)
```

Full detail: `memory-bank/architecture.md`.

## Proposed technology stack

Inherited from HRestroSewa, where it has been running in production. See `memory-bank/decisions.md`
(D-01 onward) for the reasoning and the alternatives considered.

- **Next.js** (App Router, Server Components, Server Actions) + **React** + **TypeScript** (strict)
- **Tailwind CSS v4**, `radix-ui`, `lucide-react`
- **Supabase**: Postgres, Auth, Storage, self-hosted, managed by **Coolify** on a VPS
- **web-push** (VAPID), **SSE** over Postgres `LISTEN/NOTIFY`, **Resend** (email over HTTPS),
  **pdf-lib** (prescriptions, receipts, reports), `pg` (migration runner, realtime listener)
- Dependency versions **pinned exactly**. HRestroSewa uses `latest`; CMediSewa deliberately does not.

## Repository layout

```
CMediSewa/
├── README.md                 ← you are here
├── memory-bank/              ← the project's source of truth (read first)
│   ├── project.md            vision, scope, users, MVP definition
│   ├── architecture.md       system design, auth, tenancy, realtime, PWA, deployment shape
│   ├── database.md           proposed data model, constraints, isolation, audit
│   ├── features.md           full feature inventory by surface and phase
│   ├── permissions.md        roles, permission keys, record-level (care-relationship) access
│   ├── ui-ux.md              design direction, navigation, booking UX
│   ├── workflows.md          end-to-end clinic and patient workflows
│   ├── deployment.md         environments, VPS, env vars, backups, monitoring
│   ├── security.md           threat model, RLS strategy, audit, Nepal compliance to verify
│   ├── roadmap.md            phases, implementation order, milestones
│   ├── decisions.md          decision log (D-xx accepted/proposed, P-xx pending)
│   └── current-task.md       where the project is right now
└── docs/
    └── hrestrosewa-reference.md   what was analysed in HRestroSewa and what is reused
```

**Application source code** will live at the repository root in the standard Next.js layout
(`app/`, `lib/`, `components/`, `supabase/migrations/`, `scripts/`), the same layout HRestroSewa
uses. Phase 0 creates it. Folders are added when a module needs them, not ahead of time.

## How the memory bank is organised

1. Start every session with `project.md` → `architecture.md` → `decisions.md` → `current-task.md`.
2. Read `database.md`, `permissions.md` or `security.md` only when the task touches data,
   access or patient information. Most clinical tasks touch all three.
3. When work finishes, edit only the affected files. Move finished work out of
   `current-task.md`. Add real architectural choices to `decisions.md` and never delete old
   entries from it.
4. Never duplicate. Structure goes in `architecture.md`, reasons in `decisions.md`, tables in
   `database.md`, access rules in `permissions.md`.
5. Every statement is marked as one of three kinds: **Confirmed** (verified in code or by the
   owner), **Assumption** (believed true, unverified) or **Recommendation** (proposed, awaiting
   review).

## Architectural reference: HRestroSewa

CMediSewa borrows HRestroSewa's proven *patterns* (multi-tenant shape, server-only data
access, local JWT verification, permission sections, SSE realtime, PWA posture, cron and
email services, migration tooling). It does **not** borrow restaurant tables, business logic
or assumptions. The two projects share no database, no deployment and no code dependency.
`docs/hrestrosewa-reference.md` records exactly what was analysed and what is reused.

HRestroSewa is at `../RestroSewa` (the sibling folder on the development machine).
