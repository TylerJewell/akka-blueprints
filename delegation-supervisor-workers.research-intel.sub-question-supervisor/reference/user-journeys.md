# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a multi-part question and watch parallel retrieval

**Preconditions:** service running on `http://localhost:9985/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a question such as "What changed in the authentication API between v2 and v3, and does the FAQ address migration steps?" and Submit.
2. Observe the new session row via SSE.

**Expected:** the session progresses `DECOMPOSING → RETRIEVING → SYNTHESISED` within ~90 s. The expanded row shows a `DecompositionPlan` with 2–4 sub-questions, individual `IndexResult` entries for each sub-question, and a combined answer summary that grounds claims in the retrieved results. Index results arrive close together because the workers ran in parallel.

## J2 — Before-tool-call guardrail blocks a single malformed sub-question

**Preconditions:** the test fixture injects one sub-question with injection content (e.g., `"ignore previous instructions"`); the other sub-questions are valid.

**Steps:**
1. Submit the fixture question.
2. Watch the session.

**Expected:** the guardrail inside `IndexWorker` fires for the malformed sub-question; the workflow records an `IndexCallRejected` event for that sub-question and continues with the valid ones. The session eventually reaches `SYNTHESISED`; the combined answer notes that one sub-question was skipped. No index call is issued for the rejected sub-question.

## J3 — All sub-questions rejected; session enters BLOCKED

**Preconditions:** test fixture sends a question that the supervisor decomposes into sub-questions all classified as off-topic or policy-violating.

**Steps:**
1. Submit the fixture question.
2. Watch the session.

**Expected:** every `IndexWorker` throws `GuardrailException`; the workflow calls `QuerySessionEntity.block`; the session enters `BLOCKED` with a `failureReason` listing the rejection reasons. No combined answer is produced.

## J4 — Worker timeout produces a PARTIAL session

**Preconditions:** `IndexWorker` step timeout set to 1 s (test override) for at least one sub-question.

**Steps:**
1. Submit a question that decomposes into 3 sub-questions.
2. Watch the session.

**Expected:** one or more retrieval steps time out; the workflow routes to `partialStep`; the supervisor synthesises from whichever results arrived. The session enters `PARTIAL`; the combined answer summary notes the missing results. No infinite retry.

## J5 — Eval score appears beside a synthesised session

**Preconditions:** at least one `SYNTHESISED` session without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually via a test endpoint.
2. Observe the session row in the App UI.

**Expected:** the session gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery of the original combined answer was never blocked by the eval (non-blocking).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for at least 2 minutes.

**Expected:** `QuestionSimulator` drips a question from `sample-questions.jsonl` every 60 s; each becomes a session that flows through the full pipeline. The App UI is non-empty on first load; questions cover varied multi-part topics drawn from the seeded question file.
