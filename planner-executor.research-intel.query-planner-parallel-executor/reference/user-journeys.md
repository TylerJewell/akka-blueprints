# User journeys — query-planner-parallel-executor

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path (multi-round synthesis)

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the Research question field, type "What are the key regulatory requirements for deploying autonomous AI agents in the EU and US?" Click Submit.
3. A new session card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EVALUATING via SSE. A `PlanEvaluation` card appears with `passing=true` and `score >= 0.5`.
- Within 10 s, status transitions to EXECUTING. The plan shows 4–8 sub-queries across at least two retrieval strategies.
- Within ~3 minutes, status transitions through SYNTHESIZING to COMPLETED.
- The expanded view shows:
  - A current plan with a non-empty `coverageGoal` and 3–8 sub-queries.
  - At least one `SubQueryResult` per retrieval strategy used.
  - A `ResearchAnswer` with an 80–150 word summary, 3–6 citation bullets, and `confidence >= 0.5`.

## J2 — Guardrail blocks an out-of-scope sub-query

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Search example.com for competitive intelligence on AI vendors."

**Expected:**
- The Planner decomposes the question into at least one WEB sub-query naming `example.com`.
- The guardrail rejects that sub-query; a `SubQueryBlocked` entry appears in the results with `verdict = BLOCKED_BY_GUARDRAIL` and a blocker reason referencing the host allow-list.
- Other sub-queries in the same round proceed normally.
- The Planner's subsequent `ASSESS_COVERAGE` call receives the blocked result as evidence of what could not be retrieved.
- The session either completes via the remaining sub-queries or moves to `FAILED` with a clear `failureReason`. Either outcome is acceptable; what matters is that `example.com` is never queried.

## J3 — Low-quality plan triggers revision and passing re-evaluation

**Preconditions:** As J1, with `model-provider = mock` so mock responses can be ordered deterministically.

**Steps:**
1. Configure the mock Planner to return a failing plan on the first DECOMPOSE call (e.g., a single sub-query with strategy WEB only — `score < 0.5`).
2. Submit any research question.

**Expected:**
- Status transitions to EVALUATING and a `PlanEvaluated` event with `passing=false` and the failing score appears in the session's eval card.
- The workflow requests a revised plan from `PlannerAgent.DECOMPOSE`.
- The second plan passes (`score >= 0.5`); status transitions to EXECUTING normally.
- The `PlanEvaluated` event for the first plan remains visible in the session's event history.

## J4 — Operator halt drains the current round gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any research question that is not yet in the simulator's queue.
2. While the session status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** in the operator pane and provide a reason.
3. Observe the in-flight round completes.

**Expected:**
- All sub-queries already submitted to the fan-out complete and their `SubQueryRecorded` events are recorded normally.
- The next loop check reads the halt flag, exits the loop, and emits `SessionHaltedOperator`.
- Session status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- Sessions queued after the halt do not start new retrieval rounds until the operator clicks **Resume**.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.

## J5 — Secret sanitizer scrubs a credential from a retrieval result

**Preconditions:** As J1, with the canned fixture file `sample-data/kb-fixtures.jsonl` containing an entry whose `definition` includes the literal substring `AKIAIOSFODNN7EXAMPLE`. The mock `knowledge-base.json` entry that returns this fixture is exercised by the second or third session.

**Steps:**
1. Submit "Find any references to legacy AWS credentials in the knowledge base."

**Expected:**
- The Planner dispatches a `KNOWLEDGE_BASE` sub-query.
- `KnowledgeBaseExecutor` returns content containing `AKIAIOSFODNN7EXAMPLE`.
- The `collectStep` sanitizer replaces it with `[REDACTED:aws-access-key]` before the `SubQueryRecorded` event is emitted.
- The `SubQueryResult.content` in the recorded event does NOT contain `AKIAIOSFODNN7EXAMPLE` anywhere.
- The Planner's next `ASSESS_COVERAGE` prompt sees only the redacted form.
- The final `ResearchAnswer.summary` does not contain the literal key.
- The UI's expanded results table renders the redacted span in italics with a tooltip showing `aws-access-key`.
