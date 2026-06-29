# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a job and watch parallel variant collection

**Preconditions:** service running on `http://localhost:9599/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a source text (e.g., "The deployment was successful.") and target language (e.g., "French"). Submit.
2. Observe the new job row via SSE.

**Expected:** the job progresses `PENDING → IN_PROGRESS → SELECTED` within ~60 s. The expanded row shows three `TranslationVariant` items (formal, informal, literal), the selected result with rationale, and a `guardrailVerdict` of `"ok"`. All three variants arrive close together because the workers ran in parallel.

## J2 — Branch timeout produces a partial selection

**Preconditions:** one branch step timeout set to 1 s (test override).

**Steps:**
1. Submit a job.
2. Watch the job row.

**Expected:** the timed-out branch does not contribute a variant. The workflow routes to `collectPartialStep`, collects the two returned variants, and passes them to the supervisor for selection. The job enters `PARTIAL`. The supervisor's rationale notes that one register was unavailable. No infinite retry.

## J3 — Guardrail blocks a policy-violating selection

**Preconditions:** Supervisor returns a `SelectionResult` with `guardrailVerdict` = `"blocked"` (test fixture or mock).

**Steps:**
1. Submit the fixture job.
2. Watch the job row.

**Expected:** `guardrailStep` detects the blocked verdict; the workflow calls `blockJob`; the job enters `BLOCKED` with a `failureReason`. The job never transitions to `SELECTED` or `PARTIAL`.

## J4 — Eval score appears beside a selected job

**Preconditions:** at least one `SELECTED` job without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the job row.

**Expected:** the job gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery of the original result was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `JobSimulator` drips a job from `translation-jobs.jsonl` every 60 s; each job flows through the full pipeline. The App UI is non-empty on first load. Jobs span at least two different target languages.
