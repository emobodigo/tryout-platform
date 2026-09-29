# SPEC — Tryout Platform (Phase 1: Backend API)

Phase 1 delivers **the backend only**: a REST API with Swagger documentation and a demo seeder. No frontend.

Vocabulary follows `CONTEXT.md`. The names in that glossary are the names used in code.

## 1. Scope

The API makes one Tryout platform work end to end:

- Master data: `Tryout Type` (CPNS, BUMN, OJK) with its `Question Group`s (TWK, TIU, TKP) and their grading rules.
- `Question Bank` for three question forms: `single_choice`, `true_false`, `matching`. Each Question carries an optional `Explanation` and optional image `Attachment`s.
- `Tryout` (a package) with a total time, a `Timer Mode`, and a `Question Quota` per group.
- `Attempt`: the question draw, timed answers, submission, a `Score` per group, and the `Passed` status.
- `History` and `Explanation`, both gated by the `Explanation Access` flag.
- Accounts: self-registration, and the roles `Participant`, `Admin`, `Superadmin`.

### 1.1 Enum values

```
QuestionForm     single_choice | true_false | matching
GradingMode      all_or_nothing | weighted
TimerMode        global | strict
TryoutStatus     draft | active
AttemptStatus    running | submitted | auto_submitted
ItemType         option | statement | pair
AttachmentTarget question | explanation
```

## 2. Out of scope for phase 1

Deliberately postponed. The design is still recorded; the code comes later.

| Postponed | Reason |
| --- | --- |
| `Scraping` module and the `Staging`/`Promotion` screens | no target site exists yet to test the extractor |
| CSV export of results | the admin result list is enough for now |
| Admin panel and the whole SolidJS frontend | phase 2 |
| Deployment files (Dockerfile, compose) | focus on the local machine first |
| Email (address verification, reset links) | an Admin resets a Participant password instead |
| Payments and paid packages | the catalogue is open to everyone |
| Per-question statistics (share of correct answers) | later |

## 3. Domain model

Hierarchy: `Tryout Type` → (`Question Group` → `Question`) and `Tryout Type` → `Tryout` → `Attempt` → `History`.

```
User               id, email (unique), passwordHash, name, role, canViewExplanation, status, timestamps
RefreshToken       id, userId, tokenHash, expiresAt, revokedAt, createdAt

TryoutType         id, slug (unique), name, description, sortOrder, active, timestamps
QuestionGroup      id, tryoutTypeId, name, sortOrder, weight, gradingMode, threshold, timestamps
                   unique: (tryoutTypeId, name)

Question           id, questionGroupId, form, text, explanation (nullable), active, timestamps
QuestionOption     id, questionId, text, sortOrder, isKey, value                    -- single_choice
Statement          id, questionId, text, sortOrder, isKey, value                    -- true_false
MatchingPair       id, questionId, leftText, rightText, sortOrder, value            -- matching
                   value is a number 0..weight, used only when gradingMode = weighted
                   single_choice needs exactly one key; true_false may hold one statement

Attachment         id, questionId, target (question|explanation), path, mime, sizeBytes, sortOrder, createdAt

Tryout             id, tryoutTypeId, slug (unique), title, description, totalTimeSeconds,
                   timerMode (global|strict), status (draft|active), timestamps
QuestionQuota      id, tryoutId, questionGroupId, count                    unique: (tryoutId, questionGroupId)

Attempt            id, tryoutId, participantId, timerMode, totalTimeSeconds, questionCount,
                   questionSeconds (strict mode), startedAt, deadlineAt,
                   status (running|submitted|auto_submitted), submittedAt, timestamps
AttemptQuestion    id, attemptId, questionId, questionGroupId, sortOrder, seconds,
                   openedAt, closedAt, locked                                 unique: (attemptId, questionId)
Answer             id, attemptQuestionId, answeredAt, updatedAt
AnswerItem         id, answerId, itemId, itemType (option|statement|pair), textValue (nullable), isCorrect
GroupScore         id, attemptId, questionGroupId, score, threshold, passed
```

## 4. Domain rules

### 4.1 Question Draw

- When an Attempt starts, the API draws `count` Questions per `QuestionQuota`, at random, from the **active** Questions of that `Question Group`.
- The API freezes the result in `AttemptQuestion`. The order is: groups in `sortOrder`, then Questions shuffled inside each group. The shuffle seed is the Attempt id, so an investigation can reproduce a draw.
- A draw short of the quota returns **422** `QUOTA_EXCEEDS_BANK`. The API checks this twice: when the Tryout becomes active, and when an Attempt starts.
- One running Attempt per Participant per Tryout. A second Attempt returns **409** `ATTEMPT_ALREADY_RUNNING` with the id of the running Attempt, so the frontend can resume it.

### 4.2 Timer Mode

The server holds the time. Closing the browser stops nothing.

- `global`: one `deadlineAt` = `startedAt + totalTimeSeconds`. The Participant moves between Questions freely.
- `strict`: `questionSeconds` = ⌊totalTimeSeconds ÷ questionCount⌋. The API adds the remaining seconds to the last Question. The budget of one Question never carries over: pressing Next early burns the rest. The Participant cannot return to an earlier Question. The Attempt ends after the last Question closes, or when its budget runs out.
- In `strict` mode the API computes progress **lazily** from the server clock: each request works out which Question is open. A Question whose budget passed without an Answer closes with a Score of 0. There is no scheduler and no cron job.
- The rule above holds when the Participant closes the browser too: the budget keeps running, and skipped Questions close with a Score of 0.
- Every Attempt ends automatically when its deadline passes. The status becomes `auto_submitted`, and the API computes the Score at that moment.

### 4.3 Grading

- `all_or_nothing`: the full `Weight` when every key is correct. Otherwise 0.
- `weighted`: the sum of the `value` of the correct items (option, statement, or pair). An Admin sets each value, and their total must not exceed the `Weight`.
- One `Score` per `Question Group`. The API compares it against that group's `Threshold`.
- `Passed` requires **every** `Question Group` to reach its own Threshold. There is no overall threshold.
- An unanswered Question scores 0 and still appears in `History` as empty.
- The API computes the Score once: on submission, or when the deadline passes. It stores the result in `GroupScore`.

### 4.4 Explanation and Explanation Access

- The API never sends an `Explanation` together with a Question during a running Attempt.
- A dedicated endpoint serves it: `GET /api/questions/:id/explanation`. Only a Participant with `Explanation Access` may call it (Admin and Superadmin always may). The call is allowed at any time, including during a running Attempt. See ADR-0004.
- Without the flag the API returns **403** `FORBIDDEN_EXPLANATION_ACCESS`.
- An Admin grants or removes `Explanation Access` in the Participant record. The flag never appears on its own.

### 4.5 History

- A Participant sees their own Attempts: the time, the `Score` per group, and the `Passed` status.
- Per Question, the Participant sees the question text, their own Answer, and the correct key.
- `Explanation` on the History screens still goes through the endpoint in section 4.4.

### 4.6 Accounts and roles

- Self-registration with email and password. No email verification.
- `Participant`: works on Tryouts and reads History. `Admin`: manages master data, the Question Bank, Tryouts, and Participants. `Superadmin`: all of that, plus Admin accounts.
- A forgotten password is reset by an Admin. A Participant can change their own password while signed in.
- An Admin can deactivate an account. A deactivated Participant is refused at login: **403** `ACCOUNT_INACTIVE`.

## 5. API contract

Prefix `/api`. Authentication uses a bearer access token. Swagger lives at `/api/docs`.

Every error returns an English code plus parameters. The Indonesian text lives in the frontend locale files.

```json
{ "error": { "code": "QUOTA_EXCEEDS_BANK", "params": { "questionGroupId": 7, "required": 30, "available": 20 } } }
```

| Area | Endpoint |
| --- | --- |
| Auth | `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`, `PATCH /auth/me/password` |
| Public | `GET /tryout-types`, `GET /tryout-types/:slug`, `GET /tryouts`, `GET /tryouts/:slug`, `GET /health` |
| Admin master data | `CRUD /admin/tryout-types`, `CRUD /admin/tryout-types/:id/question-groups` |
| Admin Question Bank | `CRUD /admin/questions` (filters: group, form, active), `POST /admin/questions/:id/attachments`, `DELETE /admin/attachments/:id` |
| Attachments | `GET /attachments/:id` (public file, no keys inside) |
| Admin Tryouts | `CRUD /admin/tryouts`, `PUT /admin/tryouts/:id/quota` |
| Admin Participants | `GET /admin/participants`, `PATCH /admin/participants/:id` (status, explanation access), `PATCH /admin/participants/:id/password` |
| Admin results | `GET /admin/tryouts/:id/attempts` |
| Participant — Attempt | `POST /tryouts/:id/attempts` (start), `GET /attempts/running?tryoutId=`, `GET /attempts/:id` (questions plus remaining time), `PUT /attempts/:id/answers`, `POST /attempts/:id/next` (strict mode), `POST /attempts/:id/submit` |
| History | `GET /me/attempts`, `GET /attempts/:id/result` |
| Explanation | `GET /questions/:id/explanation` |

`GET /attempts/:id` returns the Questions **without keys and without explanations**. It also returns `remainingSeconds` in `global` mode, or `remainingQuestionSeconds` plus `activeQuestionNumber` in `strict` mode.

## 6. Technical

- npm workspaces monorepo. `apps/api` holds the NestJS 11 application. `apps/web` (SolidJS) follows in phase 2.
- Drizzle ORM (`drizzle-orm` 0.45 and `drizzle-kit` 0.31, driver `mysql2` 3.24). Database `tryout_platform` on the XAMPP MariaDB (`127.0.0.1:3306`, user `root`, no password). See ADR-0001 and ADR-0005. The schema is TypeScript under `apps/api/src/db/schema/`. `drizzle-kit generate` writes SQL migration files, and those files are committed. A separate database `tryout_platform_test` holds the test data.
- Configuration through `@nestjs/config` and `.env`, with a `.env.example` beside it. Validation through `class-validator` and a global `ValidationPipe` (whitelist and transform).
- Uploads: multer disk storage into `storage/attachments/`, limit 2 MB, mime `image/jpeg`, `image/png`, or `image/webp`. Served by `GET /api/attachments/:id`.
- Tokens: access token 15 minutes, refresh token 7 days with rotation. The API stores the refresh token as a hash, so a logout can revoke it.
- Time: every timestamp is UTC in the database. Every duration is in seconds. Display time (WIB) is the frontend's job.
- The application injects its own clock provider, so a test can move time forward and check a deadline without waiting.
- Tests: Jest for units and supertest for end-to-end runs, against `tryout_platform_test`. The seeder prepares the demo data.

## 7. Work plan

`docs/TICKETS.md` holds the 11 slices in dependency order. GitHub issues #1 to #11 carry the same slices with native blocking links. Every slice has the label `ready-for-agent`.

## 8. Closed decisions

1. **`strict` mode with the browser closed** — the budget keeps running, and skipped Questions close with a Score of 0 (section 4.2). This is deliberate: it matches the "the leftover budget burns" rule, and a Participant cannot escape the time pressure by closing the browser.
2. **Database credentials** — `root` with no password (the XAMPP default) on the local machine. A dedicated user follows at deployment time.
3. **`true_false` with one statement** — a Question with exactly one `Statement`. There is no separate question form for it.
