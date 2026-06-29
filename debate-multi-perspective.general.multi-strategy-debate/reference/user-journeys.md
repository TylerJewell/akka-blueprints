# User journeys — multi-strategy-workflow

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on declared port (9155); valid model-provider API key set (or mock LLM selected).

**Steps:**
1. Open `http://localhost:9155/`. App UI tab is visible.
2. In the Question textarea type "What is the difference between a compiled and an interpreted programming language?" Click Submit.
3. A new query card appears with status RECEIVED.

**Expected:**
- Within 1 s, status transitions to RUNNING via SSE.
- Within 60 s, status transitions to SYNTHESIZED.
- The expanded view shows three strategy-result blocks (KEYWORD, SEMANTIC, CHAIN_OF_THOUGHT), each with a direct answer, a confidence value, and 2–4 evidence items; and one synthesized-answer block with a direct answer, a 60–120 word summary, and `guardrailVerdict: "ok"`. No failure reason.

## J2 — Input guardrail rejection

**Preconditions:** As J1.

**Steps:**
1. Submit a question that matches a prohibited-content pattern — for example, an empty string or a question over 2000 characters — or one whose content matches the `multistrategy.guardrail.blocked-patterns` list.

**Expected:**
- Query enters REJECTED without any strategy agent being called.
- `GET /api/queries/{id}` returns `status: "REJECTED"` and a `failureReason` naming the rejection cause (e.g. "empty question" or "prohibited content: <matched pattern>").
- No RUNNING transition fires. No strategy results are attached.
- The query card shows the REJECTED status in grey and displays the failure reason.

## J3 — Degraded strategy

**Preconditions:** As J1, plus one strategy agent's step timeout set to 1 s (configurable via env var `KEYWORD_TIMEOUT_MS=1000` or by editing `application.conf`).

**Steps:**
1. Submit any valid question.

**Expected:**
- Query progresses RECEIVED → RUNNING → DEGRADED.
- The expanded view shows the two strategy agents that returned populated results, the timed-out strategy block empty, the synthesized-answer summary acknowledging the missing strategy, and `failureReason: "strategy timeout: KeywordSearchAgent"`.
- The synthesized answer is reconciled from the available strategies only.

## J4 — Output guardrail block

**Preconditions:** As J1, plus an override that forces the coordinator's synthesized answer to fail the structural vetter (e.g., submit the question "test-output-block", which the deterministic vetter is configured to reject — see `application.conf` test-mode override).

**Steps:**
1. Submit the question "test-output-block".

**Expected:**
- Query progresses RECEIVED → RUNNING → BLOCKED.
- The expanded view shows all three strategy results still populated (visible for audit) but the synthesized-answer block carries `guardrailVerdict: "blocked: <reason>"` and the query is in BLOCKED status.
- `failureReason` names the guardrail failure.

## J5 — Cross-strategy agreement scoring

**Preconditions:** At least one SYNTHESIZED query exists with no `agreementScore`. The `EvalSampler` schedule reduced to 30 s for the test (`EVAL_SAMPLER_SECONDS=30`).

**Steps:**
1. Submit any valid question; wait for SYNTHESIZED.
2. Wait 30 s.

**Expected:**
- The query card shows an agreement-score chip (1–5) and the rationale is visible in the expanded view.
- `GET /api/queries/{id}` includes `agreementScore` and `agreementRationale` populated.
- The query's status is unchanged (still SYNTHESIZED); only the score fields are added.
- The SSE stream emits a `query-update` event for the `AgreementScored` transition.

## J6 — Background load from the simulator

**Preconditions:** Service running; no UI interaction.

**Steps:**
1. Open the App UI tab and wait.

**Expected:**
- Without any submission, a new query appears roughly every 60 s, seeded from `sample-events/query-submissions.jsonl`, and runs the full RECEIVED → RUNNING → SYNTHESIZED path on its own. At least two seeded questions exercise different domains (e.g., science and history) to show the strategy agents producing varied evidence.
