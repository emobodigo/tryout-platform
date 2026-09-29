# Phase 1 Tickets — Backend API

Vertical slices of `docs/SPEC.md`: each ticket cuts through schema → API → tests, so it can be proved on its own without waiting for another ticket.

**Status:** published as GitHub issues #1–#11 with the label `ready-for-agent` in `emobodigo/tryout-platform`, with native dependency links (blocked-by/blocking). Implementation has deliberately not started.

| Ticket | Issue | Ticket | Issue |
| --- | --- | --- | --- |
| T1 | #1 | T7 | #7 |
| T2 | #2 | T8 | #8 |
| T3 | #3 | T9 | #9 |
| T4 | #4 | T10 | #10 |
| T5 | #5 | T11 | #11 |
| T6 | #6 | | |

Ticket wording uses the English names from `docs/SPEC.md` section 1.1, which map to the terms in `CONTEXT.md`.

---

## T1 — API and database foundation
**What it delivers:** an npm workspaces repo with a live `apps/api` (NestJS 11), connected to Drizzle ORM + MariaDB through the `mysql2` driver, with Swagger documentation, a clock provider a test can fake for deadline checks, and a separate test harness on the `tryout_platform_test` database.
**Blocked by:** none (can start immediately)
- [ ] `npm run start:dev` starts the API, and `GET /api/health` answers 200
- [ ] Swagger opens at `/api/docs`
- [ ] `npm run db:generate` then `npm run db:migrate` build the schema in `tryout_platform`; `.env.example` is complete
- [ ] `npm run test` and `npm run test:e2e` pass against the test database

## T2 — Accounts, roles, and authentication
**What it delivers:** a Participant can register and sign in; an Admin can manage Participant accounts; protected endpoints refuse callers without the right.
**Blocked by:** T1
- [ ] Registration and login return an access token (15 minutes) and a refresh token (7 days, rotating)
- [ ] Logout revokes the refresh token, and the old token stops working
- [ ] `GET /api/auth/me` shows the role and the explanation access flag
- [ ] A Participant changes their own password; an Admin resets a Participant password
- [ ] `/api/admin/*` refuses a Participant (403); only a Superadmin manages Admin accounts
- [ ] A deactivated account is refused at login (`ACCOUNT_INACTIVE`)
- [ ] Seeder: 1 Superadmin, 1 Admin, 2 Participants (one with Explanation Access)

## T3 — Master data: Tryout Types and Question Groups
**What it delivers:** an Admin builds the catalogue — `Tryout Type` (CPNS/BUMN/OJK) with its `Question Group`s (TWK/TIU/TKP) carrying `Weight`, `Grading Mode`, and `Threshold`; the public reads it without signing in.
**Blocked by:** T2
- [ ] CRUD `Tryout Type` (unique slug, sort order, active) by an Admin
- [ ] CRUD `Question Group` inside one `Tryout Type`; the name is unique per type
- [ ] `GET /api/tryout-types` and `/:slug` work without a token and list active records only
- [ ] A `Threshold` change demonstrably affects Attempt grading (proved in T7)
- [ ] Demo data: CPNS with TWK (all_or_nothing, weight 5, threshold 25), TIU (all_or_nothing, 5, 30), TKP (weighted, 5, 30)

## T4 — Question Bank: three question forms
**What it delivers:** an Admin fills the Question Bank for all three forms, with keys, `Explanation`, and per-item values for `weighted` mode; a Question can be retired without damaging earlier Attempts.
**Blocked by:** T3
- [ ] `single_choice`: options, exactly one key
- [ ] `true_false`: one or more statements, each with a key
- [ ] `matching`: left–right pairs
- [ ] Per-item value (option, statement, pair) is used only when `Grading Mode` = `weighted`, and the total does not exceed the `Weight`
- [ ] `Explanation` is optional; `active = false` removes a Question from future draws
- [ ] The Question list filters by `Question Group`, form, and active status
- [ ] Validation refuses a Question with no key, no options, or an empty pair

## T5 — Image attachments
**What it delivers:** a Question and an Explanation can hold images; each image opens through its own URL without leaking a key.
**Blocked by:** T4
- [ ] Upload accepts `image/jpeg|png|webp` up to 2 MB and refuses anything else with 415/413
- [ ] An Attachment hangs on a Question or on an Explanation, and holds a sort order
- [ ] `GET /api/attachments/:id` serves the file
- [ ] Deleting an Attachment also removes the file in `storage/`
- [ ] Files live outside the application repo, so they never reach a commit

## T6 — Tryouts and Question Quota
**What it delivers:** an Admin builds a `Tryout` package with a total time, a `Timer Mode`, and a `Question Quota`; the public lists active packages; activating a package whose Question Bank is too small fails with a clear error.
**Blocked by:** T4
- [ ] CRUD `Tryout` (unique slug, `totalTimeSeconds`, `timerMode`, status `draft|active`)
- [ ] `PUT /api/admin/tryouts/:id/quota` sets the count per `Question Group`
- [ ] Activating a Tryout whose quota exceeds the active Questions fails with `422 QUOTA_EXCEEDS_BANK` plus the available count
- [ ] `GET /api/tryouts` and `/:slug` list active packages only, and include the quota and the total time
- [ ] Demo data: Paket 1 (global, 45 minutes, 10/10/10) and Paket 2 (strict, 45 minutes, 10/10/10)

## T7 — Attempt engine: `global` mode
**What it delivers:** one full Attempt through the API — start, a frozen question draw, answers, submission, and the resulting Score and `Passed` status.
**Blocked by:** T6
- [ ] Starting an Attempt draws the `Question Quota` from active Questions, orders by `Question Group` (shuffled inside a group), and freezes the result
- [ ] `GET /api/attempts/running?tryoutId=` returns the running Attempt instead of an error
- [ ] Starting a second Attempt for the same Tryout returns `409 ATTEMPT_ALREADY_RUNNING`
- [ ] Question delivery never carries a key or an Explanation
- [ ] Answer saving is idempotent and repeatable for the same Question
- [ ] A Participant may submit early; an empty Question scores 0
- [ ] An Attempt past its deadline becomes `auto_submitted` and its Score is computed
- [ ] The Score per `Question Group` is correct in `all_or_nothing` and `weighted` mode; `Passed` needs every threshold
- [ ] An Attempt owned by another Participant returns 403
- [ ] Closing the browser and returning before the deadline keeps the remaining time correct (with a faked clock in the test)

## T8 — Attempt engine: `strict` mode
**What it delivers:** an Attempt with a budget per Question — the budget runs out and the next Question opens by force, no going back, and the leftover budget burns when Next is pressed early.
**Blocked by:** T7
- [ ] Question budget = ⌊total ÷ question count⌋ seconds, with the remainder added to the last Question
- [ ] The open Question is computed from the server clock; an expired budget closes that Question with a Score of 0 with no scheduler
- [ ] Pressing Next early opens the next Question with a full budget (the burnt remainder never piles up)
- [ ] A request that touches a closed Question returns `423 QUESTION_LOCKED`
- [ ] The Attempt ends after the last Question closes or its budget runs out, and the Score is computed then
- [ ] Tests use 2700 seconds ÷ 30 Questions = 90 seconds, plus a case that does not divide evenly (for example 5390 ÷ 110, remainder added to the last Question)

## T9 — History and results
**What it delivers:** a Participant sees earlier Attempts with per-Question detail and their best Score; an Admin sees the result list of one Tryout.
**Blocked by:** T7
- [ ] `GET /api/me/attempts` carries the time, the Score per `Question Group`, and the `Passed` status
- [ ] `GET /api/attempts/:id/result` carries the question text, the Participant Answer, and the correct key
- [ ] The best Score per Tryout is marked in the History list
- [ ] `GET /api/admin/tryouts/:id/attempts` carries the Attempts of every Participant
- [ ] The History of another Participant returns 403 (an Admin may read it)

## T10 — Explanation and Explanation Access
**What it delivers:** an Explanation opens at any time for a Participant with the right — including during a running Attempt — and stays closed for everyone else (ADR-0004).
**Blocked by:** T7
- [ ] `GET /api/questions/:id/explanation` returns the Explanation and its attachments to a Participant with Explanation Access
- [ ] Without the flag it returns `403 FORBIDDEN_EXPLANATION_ACCESS`
- [ ] A Question with no Explanation returns `404 QUESTION_EXPLANATION_NOT_FOUND`
- [ ] Admin and Superadmin always succeed
- [ ] The call works in the middle of a running Attempt
- [ ] Removing Explanation Access closes the door on the next request

## T11 — Demo seeder, end-to-end proof, README
**What it delivers:** one command that turns an empty database into a ready platform, plus proof that both timer modes really run from outside.
**Blocked by:** T8, T9, T10
- [ ] `npm run seed` is idempotent: CPNS (TWK/TIU/TKP), 30 Questions with Explanations (10/10/10), Paket 1 and Paket 2, 1 Superadmin, 1 Admin, 2 Participants
- [ ] `npm run demo` runs one full Attempt in `global` mode and one in `strict` mode over HTTP, then prints the Score per `Question Group` and the `Passed` status
- [ ] README: steps from zero (start the XAMPP MySQL, migrate, seed, run, open Swagger) plus the environment variable list
- [ ] `npm run test` and `npm run test:e2e` pass from an empty database
