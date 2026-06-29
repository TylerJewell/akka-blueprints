# User journeys — financial-research-pipeline

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Full pipeline convergence

**Preconditions:** Service running on port 9767; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9767/`. App UI tab is visible.
2. In the Research query field, type "Compare the free cash flow profiles of the three largest US semiconductor companies over the last four quarters." Set sector to `equity`, leave word ceiling at 800. Click Submit.
3. A new report card appears with status `PLANNING`.

**Expected:**
- Status transitions: `PLANNING` → `RESEARCHING` → `ANALYSING` → `WRITING` → `VERIFYING` → `AWAITING_COMPLIANCE`.
- Each transition is visible in the live list via SSE within a few seconds of the previous one completing.
- When `AWAITING_COMPLIANCE` is reached, the compliance queue panel appears with Approve and Reject buttons.
- Clicking Approve transitions the report to `APPROVED`.
- The terminal block shows the approved text and "best of N drafts."
- The expanded view shows the plan (sub-tasks), per-draft timeline (guardrail verdict + verifier verdict + notes), and the sanitiser log.

## J2 — Halt at verify ceiling

**Preconditions:** As J1, plus an override that forces the Verifier to always return `REVISE` (test mode — submit the literal query `"test-force-reject"`, which the mock provider's seed logic always answers with `REVISE`).

**Steps:**
1. Submit the query `"test-force-reject"` with sector `general` and default word ceiling.

**Expected:**
- Report progresses through `PLANNING` → `RESEARCHING` → `ANALYSING` → `WRITING` → `VERIFYING` → `WRITING` → `VERIFYING` → `WRITING` → `VERIFYING` for `maxVerifyAttempts` cycles (default 3).
- After the 3rd cycle ends in `REVISE`, the report transitions to `REJECTED_FINAL`.
- The terminal block shows the highest-scoring draft's text as "best of 3 drafts" and the `rejectionReason` reads `"max verify attempts reached (3)"`.
- All 3 drafts are present in the expanded view with their guardrail and verifier verdicts.
- `GET /api/reports/{id}` returns the full Report with all 3 drafts in `drafts[]`, all 3 verifications in `verifications[]`, and `status: "REJECTED_FINAL"`.

## J3 — Guardrail short-circuit

**Preconditions:** As J1.

**Steps:**
1. Submit the query `"Provide a detailed analysis of global commodity markets across energy, metals, and agriculture sectors"` with sector `commodities` and word ceiling **300**.

**Expected:**
- The report reaches `WRITING`. The writer's first draft exceeds 300 words.
- The guardrail records `passed = false`, `reasonCode = "OVER_WORD_CEILING"`, with detail naming the word count and ceiling.
- The Verifier is NOT called for draft 1. The report stays in `WRITING`.
- The Writer is called again with structured feedback (`"Draft exceeds the configured word ceiling; condense and resubmit."`). Draft 2 is shorter and passes the guardrail.
- The verifier scores draft 2 normally. The loop continues until `APPROVE` or the retry ceiling.
- The expanded view shows draft 1 with the red `OVER_WORD_CEILING` pill (no verifier verdict row), draft 2 with `OK`, and so on.

## J4 — Sanitiser active

**Preconditions:** Service configured with at least one entry in `financial-research-pipeline.sanitiser.disallowed-patterns` (e.g., the regex `\\bXYZ:\\$\\d+(?:\\.\\d+)?\\b` to match a made-up ticker pattern).

**Steps:**
1. Submit a query whose source material the mock provider injects with a string matching the disallowed pattern (e.g., `"XYZ:$42.50"` in an excerpt).

**Expected:**
- When the report reaches `ANALYSING`, the sanitiser step runs and finds the disallowed string in an `AnalysisSection.claims` entry.
- `SanitizerLogRecorded` shows `changed = true` and lists `"XYZ:$42.50"` as a removed item.
- The App UI's sanitiser log panel shows the removed item.
- The writer's draft does not contain the string `"XYZ:$42.50"`.
- `GET /api/reports/{id}` returns `sanitizerLog.changed = true` and `sanitizerLog.removedItems = ["XYZ:$42.50"]`.

## J5 — HOTL compliance queue

**Preconditions:** At least one report has reached `AWAITING_COMPLIANCE` (any J1-style run satisfies this).

**Steps:**
1. Identify a report with `status = AWAITING_COMPLIANCE` in the live list.
2. Expand the report card. The compliance queue panel is visible with Approve and Reject buttons.
3. Click Approve.

**Expected:**
- The report transitions to `APPROVED`. The SSE stream delivers the update and the live list updates.
- The compliance queue panel disappears. The terminal block shows the approved text.
- `GET /api/reports/{id}` returns `status: "APPROVED"` and `finishedAt` populated.
- Now click Reject on a different `AWAITING_COMPLIANCE` report, supplying a reason string.
- That report transitions to `REJECTED_FINAL` with `rejectionReason` equal to the supplied string.
