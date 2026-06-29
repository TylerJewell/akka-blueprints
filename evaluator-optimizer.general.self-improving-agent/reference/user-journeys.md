# User journeys — self-improving-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Revision applied on or before the ceiling

**Preconditions:** Service running on port 9223; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9223/`. App UI tab is visible.
2. In the Task instruction field, type "Explain the difference between stateful and stateless services in two sentences." Leave the expected hint empty. Click Submit.
3. A new config card appears with status `IMPROVING`.

**Expected:**
- Within 5 s, status transitions to `IMPROVING` and the first cycle begins.
- The optimizer proposes a revision within 90 s of the workflow starting.
- The attestation step runs; on a passing verdict, the config transitions to `APPLIED` and the terminal block shows the applied system prompt and "applied after N cycles".
- The expanded view shows every cycle's proposal text, the rationale, and the attestation verdict with score delta.
- `GET /api/configs/{id}` returns the full `AgentConfig` with all cycles in `cycles[]` and `status: "APPLIED"`.

## J2 — Halt at revision ceiling

**Preconditions:** As J1, plus an override that forces the attestation gate to always fail (submit the literal instruction prefix `"test-force-reject:"`, which the mock provider's seedFor logic always answers with a failing regression result).

**Steps:**
1. Submit the instruction `"test-force-reject: summarize this topic"` with no expected hint.

**Expected:**
- Config progresses `IMPROVING` through `maxRevisions` cycles (default 3), each ending in a failed attestation.
- After the 3rd cycle fails, the config transitions to `REVISION_REJECTED` (not stuck in `IMPROVING`).
- The terminal block shows the highest-scoring proposed revision's text as "best of 3 proposals" and `rejectionReason` reads `"max revisions reached (3)"`.
- All 3 cycles are present in the expanded view, each with its proposal, rationale, and `FAILED` attestation verdict.
- `GET /api/configs/{id}` returns `status: "REVISION_REJECTED"` with all 3 cycles in `cycles[]`.

## J3 — Eval-event timeline

**Preconditions:** At least one config has completed (any terminal state).

**Steps:**
1. Click the config card to expand.

**Expected:**
- The timeline shows one `EvalRecorded` event per completed revision cycle, with `attestationResult`, `scoreDelta`, and `applied` populated.
- The terminal transition (`RevisionApplied` or `RevisionRejected`) is also surfaced as a final `EvalRecorded` event carrying the loop-level outcome.
- `GET /api/configs/{id}` includes the eval events in the response (through the SSE-projected config row) without requiring a separate fetch.

## J4 — Monitor stream receives terminal transitions

**Preconditions:** Service running on port 9223.

**Steps:**
1. Open a second browser tab or `curl` client and connect to `http://localhost:9223/api/configs/monitor/sse`.
2. In the App UI tab, submit a task and wait for it to reach a terminal state (`APPLIED` or `REVISION_REJECTED`).

**Expected:**
- Within 5 s of the terminal transition, the monitor stream emits a `revision-applied` or `revision-rejected` SSE event with the full payload (`configId`, timestamp, `cycleNumber` or `totalCycles`, score delta or rejection reason).
- No intermediate `IMPROVING` events appear on the monitor stream.
- The monitor connection stays open; a second task reaching a terminal state also appears without reconnecting.
- The App UI tab's monitor badge reads "Monitor connected" in green while the SSE connection is active.

## J5 — Concurrent task submissions

**Preconditions:** Service running on port 9223; simulator active (dripping tasks every 60 s).

**Steps:**
1. Submit three tasks in rapid succession (within 5 s of each other) via the App UI form.
2. Observe the config list.

**Expected:**
- Each task is enqueued to `TaskQueueEntity` with a unique `taskId`.
- The `TaskResultConsumer` accumulates submissions; once the batch threshold is reached, a single `ImprovementWorkflow` starts for the current active config.
- Duplicate submissions within the 10 s dedup window return the same `taskId` instead of starting a second workflow.
- The live config list shows the config in `IMPROVING` status with the cycle counter incrementing as the workflow progresses.
