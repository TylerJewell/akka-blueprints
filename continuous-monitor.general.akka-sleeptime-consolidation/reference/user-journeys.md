# User journeys — sleeptime-consolidation

## J1 — Consolidation cycle completes

**Preconditions:** Service running on port 9574; valid model-provider API key set (or mock LLM selected); `ConsolidationTrigger` enabled with threshold N=5.

**Steps:**
1. Open `http://localhost:9574/` → App UI tab.
2. Create a memory block via the "Add content" button or `POST /api/memory/blocks`.
3. Increment the step counter 5 times via `POST /api/memory/sessions/{sessionId}/step` or by adding content 5 times.
4. Wait up to 30 s for `ConsolidationTrigger` to fire.

**Expected:**
- Block transitions from ACTIVE → CONSOLIDATING → CONSOLIDATED in the left panel.
- The HOTL activity stream shows `ConsolidationStarted` followed by `BlockConsolidated` events.
- The consolidated text and change rationale appear in the right-column detail pane.
- Version number increments by 1.

## J2 — Guardrail blocks an unauthorized write

**Preconditions:** Service running; at least one ACTIVE or CONSOLIDATED block exists.

**Steps:**
1. Call `writeMemoryBlock` directly outside a `ConsolidationWorkflow` context (e.g., via a test client that does not hold a valid lease token).

**Expected:**
- The call is rejected with a structured error response.
- A `GuardrailTriggered` event is emitted; it appears as a red row in the HOTL activity stream within 1 s.
- The block status is unchanged.
- The `lastGuardrailTrip` field on the block is populated with `callerId`, `toolName`, `rejectionReason`, and `triggeredAt`.

## J3 — HOTL stream delivers events in real time

**Preconditions:** Service running; App UI open on the App UI tab.

**Steps:**
1. Open the SSE stream at `GET /api/memory/sse` in a separate browser tab or via `curl -N`.
2. Add content to an existing block.
3. Trigger a consolidation cycle (see J1).

**Expected:**
- `BlockUpdated` event arrives in the SSE stream within 1 s of the `POST /blocks/{blockId}/content` call.
- `ConsolidationStarted`, `BlockConsolidated` events arrive in the SSE stream within 1 s of each workflow step completing.
- The App UI's HOTL activity stream reflects these events without page refresh.

## J4 — Drift score appears after consolidation

**Preconditions:** At least one CONSOLIDATED block exists with no `driftScore`. `DriftEvalRunner` interval reduced for the test (`DRIFT_EVAL_SECONDS=60`).

**Steps:**
1. Complete a consolidation cycle (J1) so a block transitions to CONSOLIDATED.
2. Wait up to 60 s.

**Expected:**
- The block card shows a drift score badge (0–100) coloured by severity.
- The right-column detail pane shows `driftRationale`, `priorSummary`, and `currentSummary`.
- The block's `drift` field in `GET /api/memory/blocks/{blockId}` is populated.

## J5 — High-drift block is flagged

**Preconditions:** A consolidation run produces a block whose meaning differs materially from its prior version (drift ≥ 70). This can be forced with a mock LLM response entry with `driftScore: 82`.

**Steps:**
1. Run a consolidation that returns a high-drift score.
2. Observe the App UI.

**Expected:**
- The block card shows a red drift badge (≥ 70).
- The HOTL activity stream shows a `DriftScored` event with `driftScore ≥ 70`.
- The block is visually distinguished in the block list (e.g., amber/red border) to prompt review.

## J6 — Primary agent reads consolidated context

**Preconditions:** At least one CONSOLIDATED block exists with non-empty `consolidatedText`.

**Steps:**
1. Call `POST /api/memory/agent/query` with `{ "sessionId": "<id>", "prompt": "What are the constraints for this session?" }`.

**Expected:**
- Response `AgentResponse.reply` references facts from the consolidated block.
- `blockIdRead` and `blockVersionRead` match the current block state.
- The reply does not invent facts not present in the consolidated text.
