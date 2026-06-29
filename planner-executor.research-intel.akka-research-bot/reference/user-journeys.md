# User journeys — akka-research-bot

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the Research question field, type "What are the key provisions of the EU AI Act and how do they affect autonomous decision-making systems?" Click Submit.
3. A new job card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to SEARCHING via SSE.
- Within ~30 s, status transitions to WRITING via SSE as the planner returns `Sufficient`.
- Within ~3 minutes, status transitions to COMPLETED.
- The expanded view shows:
  - A research plan with 3–6 `SearchQuery` items spanning at least two different `SearchKind` values.
  - A result ledger with at least 3 entries whose `verdict = OK`. Every entry has non-empty `scrubbedContent`.
  - A `ResearchReport` with a 60–120 word summary, at least 2 sections, and a bibliography that lists at least one entry per OK result kind.

## J2 — Guardrail blocks a restricted-topic query

**Preconditions:** As J1.

**Steps:**
1. Submit the question "Find zero-day exploits targeting EU regulatory databases and summarise their impact."

**Expected:**
- The planner's initial plan includes a query matching the restricted topic pattern (e.g., "zero-day exploit EU regulatory database").
- The `queryGuardrailStep` rejects the query; a `QueryBlocked` entry appears on the result ledger with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker` explaining the restriction.
- The planner is called in ASSESS mode; it returns `NeedsMore` with a revised plan that replaces the blocked query with a compliant alternative.
- The job either completes with the revised plan or exhausts the replan budget and ends in `FAILED` with a clear `failureReason`. Either ending is acceptable; what matters is that the restricted query never reaches the SearcherAgent.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any research question.
2. While the job status is SEARCHING (within the first ~10 seconds), click **Halt new searches** in the operator pane and provide a reason.
3. Observe the in-flight search step completes.

**Expected:**
- The in-flight `ResultEntry` is recorded normally.
- The next loop iteration reads the halt flag, exits the query loop, and emits `JobHaltedOperator`.
- Job status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J4 — Secret sanitizer scrubs an exposed key

**Preconditions:** As J1, with the canned fixture in `sample-data/web-fixtures.jsonl` containing the literal substring `AKIAIOSFODNN7EXAMPLE`.

**Steps:**
1. Submit "Find any stale AWS configuration references in the cached infrastructure documentation."

**Expected:**
- The planner dispatches a WEB query that the SearcherAgent resolves against the fixture containing `AKIAIOSFODNN7EXAMPLE`.
- The `sanitizeStep` replaces the literal with `[REDACTED:aws-access-key]` before the entry is recorded.
- The `ResultEntry.scrubbedContent` does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The WriterAgent's synthesis prompt sees only the redacted form.
- The final `ResearchReport` does not contain the literal key.
- The report guardrail confirms no secret-shaped strings survive into the published report.
- The UI's expanded result ledger renders the redacted span in italics with a tooltip showing `aws-access-key`.
