# User journeys — composed-hiring-workflow

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: application to extended offer

**Preconditions:** Service running on port 9828; a model-provider key set, or the mock LLM selected during scaffolding.

**Steps:**
1. Open `http://localhost:9828/`. The App UI tab is visible.
2. In the form, enter candidate name "Alice Chen", job role "Senior Software Engineer", and a short CV text. Click Submit.
3. An `applicationId` is returned; the application appears with status `SUBMITTED`.

**Expected:**
- The hiring manager opens a brief; the application moves to `OPENED` and the brief role summary appears on the card.
- `CandidateWorkflow` starts: the screening lead plans dimensions, screener instances each write a note, and the lead synthesises a screening report; the application moves through `SCREENING` to `SCREENED` and the report summary appears.
- `CvImprovementLoop` starts: the CV coach produces a revised draft, the CV critic scores it and returns `PASS` (or the loop runs up to three iterations); the application moves through `CV_IMPROVING` to `CV_IMPROVED` and the iteration count appears.
- The interview panel runs: three interviewer instances score the candidate on `technical`, `behavioural`, and `cultural`; with all axes `PROCEED` the verdict is `PROCEED`, the application moves through `INTERVIEWING` to `SHORTLISTED`.
- The hiring manager drafts an offer letter; the output guardrail passes; the application moves through `OFFER_PENDING` to `OFFER_EXTENDED` with an offer reference.
- Three stage-eval score chips (screening / cv_improvement / interview) appear on the card.
- Both boards update live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-scoring under contention

**Preconditions:** As J1, with at least two screener instances and at least one open dimension slot.

**Steps:**
1. Submit an application so `CandidateWorkflow` plans several dimensions and they land simultaneously.

**Expected:**
- Each dimension shows exactly one screener's note; no dimension is evaluated by two screeners.
- Screeners that do not win the assignment return without writing a duplicate note.
- The screening report reflects one note per dimension.

## J3 — Bounded re-assessment loop

**Preconditions:** As J1. Under the mock LLM, the `interviewer.json` set includes at least one `REASSESS` score; with a real model, a CV that has a genuine gap on one competency axis.

**Steps:**
1. Submit an application whose panel returns a `REASSESS` verdict on one axis.
2. Watch the application.

**Expected:**
- The panel verdict is `REASSESS`; the named competency appears in `reassessCompetencies`.
- The application returns to `CV_IMPROVING` with `reassessCount` now 1; `CvImprovementLoop` runs again.
- After the second CV improvement round, the application proceeds to interview again and advances to offer on the second pass (a second `REASSESS` is accepted rather than looping again), so the pipeline terminates.

## J4 — Refused nested-workflow invocation

**Preconditions:** As J1. Under the mock LLM, a `screener.json` entry whose note targets a mismatched `applicationId`; with a real model, a guardrail condition triggered by the application being in the wrong state.

**Steps:**
1. Submit the application; watch the offending stage.

**Expected:**
- The before-agent-invocation guardrail refuses the `ApplicationTools` write; it never lands.
- The stage is recorded with the guardrail reason; the agent returns without writing to the workspace.
- No note or score is written into an application the agent was not assigned, and no write lands on an application already in a terminal state.

## J5 — Post-hire compliance review

**Preconditions:** An application has reached `OFFER_EXTENDED` (run J1 first).

**Steps:**
1. On the extended-offer card, open the post-hire review box.
2. Enter a reviewer id, pick `FLAGGED`, add a comment, and Submit (or `POST /api/applications/{id}/post-hire-review`).

**Expected:**
- The review is recorded against the application; the card shows the recorded `FLAGGED` outcome and comment.
- The application's status stays `OFFER_EXTENDED` — the review does not change it.
- Posting a post-hire review to an application that is not yet `OFFER_EXTENDED` returns `409` and records nothing.
