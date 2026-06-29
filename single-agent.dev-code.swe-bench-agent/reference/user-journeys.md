# User journeys — swe-bench-agent

## J1 — Submit a seeded task and get a passing patch

**Preconditions:** Service running on declared port (`http://localhost:9219/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9219/` → App UI tab.
2. From the **Task** dropdown, pick `Python AttributeError`.
3. Click **Load seeded example** to fill the bug description and repository name.
4. Click **Submit task**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SNAPSHOT_PREPARED` within 1 s. The right-pane detail shows the snapshot file tree; test files are highlighted.
- Within 60 s the card reaches `PATCH_PRODUCED`. The diff viewer shows the unified diff and the confidence score bar.
- The card transitions to `GATE_PASSED` within 10 s. The test case table shows all tests as PASS with a runtime under 1 s.
- The card reaches `COMPLETED`. The gate verdict badge shows green PASS.

## J2 — Test gate blocks a failing patch and retries

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `patch-issue.json` includes a FAILING-GATE entry — a structurally valid patch that produces test failures when applied.

**Steps:**
1. Submit any seeded task three times in a row (J1 steps × 3).
2. On the third submission, watch the gate verdict in the card detail and the network panel (`/api/tasks/sse`).

**Expected:**
- The third submission's first patch iteration produces the FAILING-GATE diff (seeded by mock determinism).
- `TestGateRunner` returns `GateVerdict.FAIL`. The card transitions to `GATE_FAILED`.
- The workflow retries `patchStep`. The attempt counter on the card increments to 2.
- The second iteration produces a passing diff. The card transitions to `GATE_PASSED` then `COMPLETED`.
- The test case table for the final gate run shows all tests as PASS.

## J3 — PatchGuardrail blocks a malformed patch before it leaves the agent

**Preconditions:** Mock LLM mode. The mock includes a MALFORMED entry with `confidenceScore = 150`.

**Steps:**
1. Submit any seeded task that the mock selects the malformed entry for (every 3rd submission by seed).
2. Watch the service log for `guardrail.reject` lines.

**Expected:**
- The agent's first iteration produces a `PatchResult` with `confidenceScore = 150`.
- `PatchGuardrail` rejects it. The malformed result NEVER lands in `BenchmarkTaskEntity` — there is no `PatchProduced` event with the malformed payload.
- The agent loop retries on iteration 2 and produces a structurally valid `PatchResult`.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming the failed check (`confidence-out-of-range`).
- The card eventually reaches `GATE_PASSED` or `GATE_FAILED` based on the retry patch's correctness.

## J4 — Secrets in the snapshot never reach the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the snapshot preparer runs either way).

**Steps:**
1. Submit a custom task whose `snapshotBytes` contains the literal string `AKKA_SECRET_KEY=sk-abc123` (simulating an accidentally committed secret).
2. Wait for `SNAPSHOT_PREPARED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/tasks/{id}` and read `snapshot.snapshotRef`.

**Expected:**
- The logged LLM call body contains `[REDACTED-SECRET]` in place of `sk-abc123`. The raw string does not appear.
- `snapshot.snapshotRef` is the SHA-256 hex of the normalized (redacted) content — not a direct hash of the raw bytes.
- `snapshot.filePaths` lists the file that contained the secret; the file appears in the diff viewer with the placeholder visible.

## J5 — Multi-attempt completion is recorded accurately

**Preconditions:** Mock LLM mode with the FAILING-GATE entry active on the first attempt.

**Steps:**
1. Submit a task that triggers the gate-failure path (same deterministic seed as J2).
2. Wait for `COMPLETED`.

**Expected:**
- `BenchmarkTask.attemptNumber` is `2` in the `GET /api/tasks/{id}` response, confirming the retry occurred.
- The `gateReport` on the entity reflects the second attempt's test run (all PASS), not the first attempt's (some FAIL).
- The card detail in the UI shows the final successful diff and gate report, with the attempt counter displaying `2`.
