# User journeys — self-improving-deep-researcher

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Convergence and memory update

**Preconditions:** Service running on port 9745; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9745/`. App UI tab is visible.
2. In the Research query field, type "regulatory approaches to large language model transparency in the EU". Leave depth at the default Standard. Click Submit.
3. A new session card appears with status `RESEARCHING`.

**Expected:**
- Within 2 s, the first attempt's executive summary stub appears in the expanded view.
- The evaluator scores the first attempt. If the verdict is `REFINE`, a second attempt is initiated automatically; this continues until either `SUFFICIENT` or the retry ceiling is reached.
- On `ACCEPTED`, the session card shows a green border and the terminal block shows the accepted report's executive summary and evidence count.
- The memory diff block appears below the attempts, listing at least one `MemoryBlockChange` with a `reason` sentence.
- `GET /api/sessions/{id}` returns the full `Session` with `status: "ACCEPTED"`, all attempts in `attempts[]`, and a populated `memoryDiff`.

## J2 — Ceiling reached, memory still updated

**Preconditions:** As J1, plus an override that forces the evaluator to always return `REFINE` (test mode — submit the literal topic `"test-force-refine"`, which the mock provider's `seedFor` logic always answers with `REFINE`).

**Steps:**
1. Submit the topic `"test-force-refine"` with depth Standard.

**Expected:**
- Session progresses `RESEARCHING` → `EVALUATING` → `RESEARCHING` → `EVALUATING` → `RESEARCHING` → `EVALUATING` for `maxAttempts` cycles (default 3).
- After the 3rd cycle ends in `REFINE`, the session transitions to `MAX_ATTEMPTS_REACHED` (not stuck in `EVALUATING`).
- The memory diff block still appears. The mock `PromptRewriterAgent` returns a diff with 3 block changes reflecting the failure pattern.
- The expanded view shows all 3 attempts, each with an executive summary, evidence count, `REFINE` verdict, and quality notes.
- `GET /api/sessions/{id}` returns `status: "MAX_ATTEMPTS_REACHED"` and a populated `memoryDiff`.

## J3 — Drift detection

**Preconditions:** At least one session has completed (any terminal state) and updated `PromptMemory`.

**Steps:**
1. Wait up to 120 s after a session completes (two `DriftSampler` ticks at 60 s each).

**Expected:**
- The `DriftSampler` detects the fingerprint change (the memory update from the completed session changed at least one block).
- A `drift-detected` SSE event appears in the App UI's drift timeline for the session.
- The drift event shows the old and new fingerprints and `changedBlockCount >= 1`.
- Subscribing to `GET /api/sessions/sse?channel=memory-updates` shows the `memory-update` event from the completed session followed by the `drift-detected` event.

## J4 — Recertification gate blocks drifted build

**Preconditions:** At least one session has run and updated `governance/prompt-memory.json` such that the block count difference vs `governance/prompt-baseline.json` exceeds `max-diff-blocks` (default 2).

**Steps:**
1. Run `mvn verify` from the project root after the memory has drifted.

**Expected:**
- `PromptBaselineVerificationTest` fails with an `AssertionError` whose message names the drifted block IDs and the measured diff count.
- The build exits non-zero.
- Updating `governance/prompt-baseline.json` to match the current memory, then re-running `mvn verify`, passes the test and exits zero.

## J5 — Memory-updates SSE channel

**Preconditions:** Service running on port 9745.

**Steps:**
1. Open a terminal and run `curl -N "http://localhost:9745/api/sessions/sse?channel=memory-updates"`.
2. In a second terminal (or the App UI), submit a research query and wait for it to complete.

**Expected:**
- When the session reaches a terminal state and `PromptRewriterAgent` runs, a `memory-update` SSE event appears in the curl stream within 5 s of the `MemoryUpdated` event being emitted.
- The event payload includes the full `MemoryDiff` with `changes[]`, `triggerSessionId`, `triggerVerdict`, and `diffedAt`.
- The regular SSE stream (without `?channel=memory-updates`) also carries the same `memory-update` event as part of the session's full update sequence.
