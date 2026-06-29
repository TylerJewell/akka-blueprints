# User journeys — parallel-voting-workflow

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on declared port (9227); valid model-provider API key set (or mock LLM selected).

**Steps:**
1. Open `http://localhost:9227/`. App UI tab is visible.
2. In the Title field type "Migrate invoicing DB"; in the Task description field paste two paragraphs describing a weekend database migration with a rollback plan. Click Submit.
3. A new task card appears with status SUBMITTED.

**Expected:**
- Within 1 s, status transitions to VOTING via SSE.
- Within 60 s, status transitions to DECIDED.
- The expanded view shows three voter blocks (Feasibility, Risk, Alignment), each with a ballot (`APPROVE`/`REJECT`/`ABSTAIN`), a 1–5 confidence score, and 2–4 weighted reasons; and one aggregated decision block with a decision (`PROCEED`/`HOLD`/`DECLINE`), a 60–100 word summary, and `quorumMet: true`. No failure reason.

## J2 — Partial aggregation from voter timeout

**Preconditions:** As J1, plus one voter's step timeout set to 1 s (configurable via env var `RISK_TIMEOUT_MS=1000` or by editing `application.conf`).

**Steps:**
1. Submit any task.

**Expected:**
- Task progresses SUBMITTED → VOTING → PARTIAL.
- The expanded view shows the two voters that returned populated, the timed-out dimension showing empty, the aggregated summary acknowledging the missing dimension, and `failureReason: "voter timeout: RiskVoter"`.
- The aggregated decision is drawn from the two available votes only.

## J3 — Inconclusive quorum

**Preconditions:** As J1, plus a test-mode override that forces the three voters to return all-distinct ballots (e.g., Feasibility = APPROVE, Risk = REJECT, Alignment = ABSTAIN). The title `"test-no-quorum"` triggers this override via the deterministic ballot fixture in `application.conf`.

**Steps:**
1. Submit the task titled "test-no-quorum".

**Expected:**
- Task progresses SUBMITTED → VOTING → INCONCLUSIVE.
- The aggregated decision block shows `quorumMet: false`.
- The status is INCONCLUSIVE, not DECIDED.
- All three voter blocks are still visible for inspection.

## J4 — Alignment eval scoring

**Preconditions:** At least one DECIDED task exists with no `alignmentScore`. The `EvalSampler` schedule reduced to 30 s for the test (`EVAL_SAMPLER_SECONDS=30`).

**Steps:**
1. Submit any task; wait for DECIDED.
2. Wait 30 s.

**Expected:**
- The task card shows an alignment-score chip (1–5) and the rationale is visible in the expanded view.
- The task's row in `/api/tasks` includes `alignmentScore` and `alignmentRationale` populated.
- The task's status is unchanged (still DECIDED); only the score fields are added.

## J5 — Background load from the simulator

**Preconditions:** Service running; no UI interaction.

**Steps:**
1. Open the App UI tab and wait.

**Expected:**
- Without any submission, a new task appears roughly every 60 s, seeded from `sample-events/task-submissions.jsonl`, and runs the full SUBMITTED → VOTING → DECIDED path on its own.
- At least one seeded task demonstrates a split vote so the quorum path is exercised in the background.
