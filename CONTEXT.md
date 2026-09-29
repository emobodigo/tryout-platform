# Tryout Platform

A public platform for working on exam tryouts (CPNS, BUMN, OJK, and similar): a tryout catalogue, a question bank, timed work sessions, and explanations that a flag unlocks.

## Language

### Catalogue

**Tryout Type**:
A catalogue classification (for example CPNS, BUMN, OJK). It owns a set of Tryouts and holds their Question Groups with the Weight, Grading Mode, and Threshold.
_Avoid_: tryout category, exam type, exam

**Tryout**:
One package a Participant works on: an ordered set of Questions with its own total time, Timer Mode, and Question Quota. It belongs to one Tryout Type.
_Avoid_: quiz, online exam, test

**Question Group**:
A subject part inside one Tryout Type (for example TWK, TIU, TKP). It owns Questions and carries the Weight, Grading Mode, and Threshold.
_Avoid_: question category, subtest, subject

**Question Quota**:
The number of Questions one Tryout takes from each Question Group (for example TWK 30, TIU 35, TKP 35). An Admin sets it.
_Avoid_: question count, item count

### Questions

**Question**:
One test item with its answer key. It belongs to exactly one Question Group, and its Explanation is optional. A Question can be retired (inactive) without changing an earlier Attempt that used it.
_Avoid_: item, prompt, point

**Item**:
One part a Participant answers inside a Question: an option (`single_choice`), a statement (`true_false`), or a pair (`matching`). In `weighted` mode an Admin gives each Item its own value.
_Avoid_: choice, element, field

**Question Form**:
The shape of the question: `single_choice` (exactly one correct answer), `true_false` (one statement alone or several statements at once), or `matching` (pairs across two columns). An Admin picks it while writing the Question.
_Avoid_: question type, question kind, form type

**Question Bank**:
The set of active Questions of one Tryout Type. It feeds the Question Draw.
_Avoid_: question pool, repository

**Staging**:
A holding area for Questions that Scraping collected and that the Question Bank has not accepted yet.
_Avoid_: draft, inbox, queue

**Scraping**:
Collecting Questions and Explanations from a web page, from a URL that an Admin pastes. It is not a scheduled crawler.
_Avoid_: crawling, harvesting, spider

**Promotion**:
The Admin action that lifts a Question from Staging into the Question Bank after editing and approval.
_Avoid_: publish, approve, conversion

**Explanation**:
The text that explains the key of one Question. Only a Participant with Explanation Access can read it, at any time, including while an Attempt runs.
_Avoid_: solution, discussion, rationale

**Attachment**:
An image attached to a Question or to an Explanation. An Admin uploads it, and the server stores it.
_Avoid_: file, media, upload

### Grading

**Grading Mode**:
How a Question Group awards points. `all_or_nothing`: every key must be correct before points appear (TWK/TIU style). `weighted`: each Item carries its own value (TKP style).
_Avoid_: scoring mode, partial scoring

**Weight**:
The points of one Question under `all_or_nothing`, or the maximum points of one Question under `weighted`. It is the same for every Question in one Question Group.
_Avoid_: score value, mark, point value

**Threshold**:
The minimum Score of one Question Group for an Attempt to pass. A pass needs **every** Question Group to reach its own threshold, and there is no overall threshold.
_Avoid_: passing grade, cut-off, minimum score

**Score**:
The points an Attempt collects in one Question Group. The API compares it against the Threshold to decide Passed.
_Avoid_: grade, mark, rating

**Passed**:
The status of an Attempt when every one of its Question Groups reaches its Threshold.
_Avoid_: passed test, success, graduate

### Attempt

**Attempt**:
One occasion on which a Participant works on a Tryout. It holds the Question Draw, the temporary Answers, the progress status, and the result. The server holds its deadline, so the Attempt survives a closed browser, and the number of Attempts per Participant has no limit.
_Avoid_: session, exam, run

**Question Draw**:
The outcome of taking the Question Quota from the Question Bank for one Attempt. The API freezes it when the Attempt starts, so the order stays fixed until the Attempt ends.
_Avoid_: random pick, sample, shuffle

**Answer**:
The Participant choice for one Question inside one Attempt. One Question has one Answer per Attempt.
_Avoid_: response, answer sheet

**Timer Mode**:
The time rule of one Tryout. `global`: one countdown for the whole Attempt, and the Participant moves between Questions freely. `strict`: each Question has its own budget (total time ÷ number of Questions) and the Participant moves by force when the budget expires, with no way back; pressing Next early burns the rest of that budget, and the Attempt ends when the last Question closes or its budget expires. An Admin picks the mode for each Tryout.
_Avoid_: timing mode, per-question timer, timer type

**History**:
The list of earlier Attempts of one Participant with their Score and Passed status. A Tryout shows the best Score from that list.
_Avoid_: log, record, archive

### Access

**Explanation Access**:
A flag on a Participant account that opens the Explanation. An Admin grants it, and it never appears by itself.
_Avoid_: premium, subscription, paid tier

**Participant**:
A user who works on Tryouts.
_Avoid_: student, member, user

**Admin**:
A user who manages Tryout Types, Questions, and Participants.
_Avoid_: operator, manager, staff

**Superadmin**:
An Admin with full authority, including the management of Admin accounts.
_Avoid_: root, owner, superuser
