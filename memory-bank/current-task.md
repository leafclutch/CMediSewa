# Current Task

**Phase:** Pre-Phase-0, architecture & planning. **No application code exists.**
**Last updated:** 2026-11-20

## Status
- Project folder: `Downloads/CMediSewa`, a separate git repo (remote
  `github.com/sanjib001/CMediSewa`), sibling to HRestroSewa (`Downloads/RestroSewa`).
  HRestroSewa was read but **not modified**.
- Documentation drafted. Every decision is **Proposed** and awaiting the owner's review.
- Nothing is committed yet. The owner drives git.

## Completed
- **HRestroSewa architecture analysis** (code-verified): stack, route groups, tenancy,
  server-only data access + RLS posture, auth (local JWT, live staff row, PIN model),
  permissions (sections/levels/presets, assignment scoping), realtime (LISTEN/NOTIFY → SSE),
  push, email (Resend/Gmail), cron (pg_cron → secret routes), storage, PWA, deployment
  (Coolify, self-hosted Supabase, migration runner, backups).
  Recorded in `../docs/hrestrosewa-reference.md`.
- **Documentation:** `README.md`, `memory-bank/` (project, architecture, database, features,
  permissions, ui-ux, workflows, deployment, security, roadmap, decisions, current-task).

## Decisions pending (see `decisions.md`)
Blocking Phase 0: **P-01** domain · **P-02** hosting (needed only by prod, but affects cost
planning) · **P-03** scheduling mode · **P-07** admin clinical access · **P-11** MFA scope.
Before go-live: **P-08** tax/IRD & Rx rules · **P-09** data residency · **P-10** retention &
consent · **P-14** pricing.
Non-blocking: P-04, P-05, P-06, P-12, P-13, P-15, P-16.

## Immediate next steps
1. Owner reviews the memory bank, especially `permissions.md` (care relationship, D-06) and
   `database.md` (the model).
2. Owner answers P-01, P-03, P-07 and P-11, and either accepts or amends D-01 to D-25.
3. Start the legal track in parallel: share `security.md` §10 with Nepali counsel.
4. Pilot clinic: identify one and run a 1-hour workflow walkthrough using `workflows.md`,
   to validate tokens vs slots, the receptionist's needs and the Rx print format.
5. Then begin **Phase 0** (`roadmap.md`), with the first implementation task being
   **"Phase 0 scaffold + migration 0001 (access posture) + CI posture check"**.

## Open questions for the owner
- Is the pilot clinic token-based or slot-based today?
- Should the existing GitHub repo be private? It will hold infrastructure docs.
