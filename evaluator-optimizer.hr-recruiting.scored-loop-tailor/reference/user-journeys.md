# User journeys — scored-loop-tailor

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9587; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9587/`. App UI tab is visible.
2. In the Job title field, type "Senior Java Engineer". In Job description, paste two to three sentences describing the role. Leave Candidate profile empty. Click Submit.
3. A new application card appears with status `TAILORING`.

**Expected:**
- Within 1 s, status transitions to `TAILORING` (already there) and the first attempt's CV draft appears.
- The guardrail verdict pill on attempt 1 reads `OK`.
- Within 60 s of submission, either:
  - Attempt 1's review is `APPROVE` and the application transitions to `APPROVED`, OR
  - Attempt 1's review is `REVISE` with 1–3 bullets and the application transitions back to `TAILORING`, with attempt 2 appearing shortly after; this continues until either an `APPROVE` or the retry ceiling is reached.
- On `APPROVED`, the terminal block shows the approved CV text and "best of N attempts."
- The expanded view shows every attempt's draft, guardrail verdict, reviewer verdict, score, and notes.

## J2 — Halt at retry ceiling

**Preconditions:** As J1, plus an override that forces the Reviewer to always return `REVISE` (test mode — submit the literal job title `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the job title `"test-force-reject"` with an empty description.

**Expected:**
- Application progresses `TAILORING` → `REVIEWING` → `TAILORING` → `REVIEWING` → … for `maxAttempts` cycles (default 4).
- After the 4th cycle ends in `REVISE`, the application transitions to `REJECTED_FINAL` (not stuck in `REVIEWING`).
- The terminal block shows the highest-scoring draft's text as "best of 4 attempts" and the `rejectionReason` reads `"max attempts reached (4)"`.
- All 4 attempts are present in the expanded view, each with its draft, guardrail OK verdict, REVISE review, and score.
- `GET /api/applications/{id}` returns the full Application with all 4 attempts in `attempts[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit on missing sections

**Preconditions:** As J1.

**Steps:**
1. Submit the job title `"Data Engineer"` with any description.

**Expected:**
- Attempt 1's draft omits the Skills section (mock provider's seedFor routes this applicationId to the incomplete mock draft). The guardrail records `verdict.passed = false`, `reasonCode = "MISSING_SECTIONS"`, with detail naming the missing section.
- The Reviewer is NOT called for attempt 1. The application stays in `TAILORING`.
- The Tailor is called again with structured feedback (`"CV is missing required sections; include Summary, Experience, and Skills before resubmitting."`). Attempt 2 contains all three sections and passes the guardrail.
- The reviewer scores attempt 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows attempt 1 with the incomplete draft and the red `MISSING_SECTIONS` pill, attempt 2 with `OK`, and so on.

## J4 — Eval-event timeline

**Preconditions:** At least one application has completed (any terminal state).

**Steps:**
1. Click the application card to expand.

**Expected:**
- The timeline shows one `ReviewEvalRecorded` event per reviewed attempt, with `verdict`, `score`, and `missingSections` populated.
- The terminal transition (ApplicationApproved or ApplicationRejectedFinal) is also surfaced as a final `ReviewEvalRecorded` event carrying the loop-level outcome.
- `GET /api/applications/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.

## J5 — Simulator populates the list

**Preconditions:** Service started at least 65 s ago; no manual submissions.

**Steps:**
1. Open the App UI tab without submitting anything.

**Expected:**
- At least one application card is visible, submitted by the `SubmissionSimulator`.
- The card's job title matches one of the entries in `src/main/resources/sample-events/job-postings.jsonl`.
- The card shows a status of `TAILORING`, `REVIEWING`, `APPROVED`, or `REJECTED_FINAL` depending on how far the loop has progressed.
