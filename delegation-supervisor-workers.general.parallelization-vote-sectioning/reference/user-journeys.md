# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a SECTIONING job and watch parallel section processing

**Preconditions:** service running on `http://localhost:9255/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Select SECTIONING mode. Enter a multi-part prompt (e.g., "Describe the history, current state, and future outlook of renewable energy."). Submit.
2. Observe the new job row via SSE.

**Expected:** the job progresses `PARTITIONING → IN_PROGRESS → AGGREGATED` within ~90 s. The expanded row shows 2–4 `SectionResult` items arriving close together (parallel execution). The aggregated answer stitches sections in index order without repetition. The `aggregationVerdict` is `"ok"`.

## J2 — Submit a VOTING job and verify vote aggregation

**Preconditions:** service running; model provider configured.

**Steps:**
1. Open the App UI tab. Select VOTING mode. Enter a question with a defensible position (e.g., "Is it better to optimize for latency or throughput in a streaming data pipeline?"). Submit.
2. Wait for the job to reach AGGREGATED.
3. Expand the row.

**Expected:** 2–4 `VoteResult` items each carry an independent `answer` and a `confidence` score. The aggregated `answer` either states the majority position or summarises the split. The supervisor does not invent a consensus that the vote results do not support.

## J3 — Worker timeout produces a PARTIAL job

**Preconditions:** one worker step timeout set to 1 s (test override).

**Steps:**
1. Submit a job (either mode).
2. Watch the job row.

**Expected:** the timed-out worker's result is absent. The workflow routes to `partialStep`, the Supervisor aggregates from available results, and the job enters `PARTIAL`. The aggregated answer notes which section or worker index is missing. No infinite retry.

## J4 — Guardrail blocks a policy-violating aggregation

**Preconditions:** Supervisor is configured via a test fixture to return an `aggregationVerdict` of `"policy_violation"`.

**Steps:**
1. Submit the fixture prompt.
2. Watch the job row.

**Expected:** `guardrailStep` detects the policy violation; the workflow calls `JobEntity.block`; the job enters `BLOCKED` with a `failureReason` describing the violation. The App UI row shows the BLOCKED status pill; the expanded row shows the failure reason. The job never shows in the AGGREGATED filter.

## J5 — Eval score appears beside an AGGREGATED job

**Preconditions:** at least one `AGGREGATED` job without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly via a test hook.
2. Observe the job row in the App UI.

**Expected:** the job acquires an `evalScore` (1–5) and an `evalRationale`. The row in the App UI shows the score. Delivery was never blocked by the eval (non-blocking). The `EvalScored` event is visible in the entity event log.
