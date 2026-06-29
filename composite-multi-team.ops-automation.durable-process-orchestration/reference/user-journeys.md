# User journeys — sk-process

Acceptance criteria. The generated system passes when all six journeys complete as written.

## J1 — Happy path: template to completed job summary

**Preconditions:** Service running on port 9440; a model-provider key set, or the mock LLM selected during scaffolding. The executor roster (`executor-1`, `executor-2`, `executor-3`) is running and idle-polling the step board.

**Steps:**
1. Open `http://localhost:9440/`. The App UI tab is visible.
2. In the form, enter the template `dependency-audit`. Click Submit.
3. A `jobId` is returned; the job appears with status `SUBMITTED`.

**Expected:**
- The coordinator plans the job; the job moves to `PLANNED` and the plan objective appears on the card.
- The step planner refines the plan; step cards appear in the step board's `Open` group and the job moves to `EXECUTING`. Executors claim steps (each claimed card shows an executor id) and move them `Open` → `Claimed` → `Done`. The job-progress line tracks the count.
- When every step is `Done`, the batch assembles and the job moves to `VALIDATING`.
- The quality panel scores the batch; with all criteria passing the verdict is `PASS`, the job moves through `VALIDATING` to `APPROVED`, the coordinator assembles the summary, and the job moves to `COMPLETED` with a result reference.
- Two step-signal score chips (batch / validation) appear on the card.
- Both boards update live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two executors idle and at least one `OPEN` step on the board.

**Steps:**
1. Submit a job so the planning desk puts several steps on the board at once.

**Expected:**
- Each step shows exactly one executor id; no step is ever shown claimed by two executors.
- Executors that lose a claim race return to polling and pick up different steps (or idle if none remain).
- The claim count per step is one.

## J3 — Bounded retry loop

**Preconditions:** As J1. Under the mock LLM, the `quality-checker.json` set includes at least one `RETRY` note naming a step name; with a real model, a batch with a real flaw on one criterion.

**Steps:**
1. Submit a job whose batch draws a `RETRY` from one of the quality criteria.
2. Watch the job.

**Expected:**
- The quality verdict is `RETRY`; the named step(s) reset to `OPEN` on the board and the job returns to `EXECUTING` with `retryCount` now 1.
- An executor claims and re-runs the reset step; the batch reassembles and the panel scores it again.
- On the second pass the job proceeds to finalise (a second `RETRY` is accepted rather than looping again), so the pipeline terminates.

## J4 — Refused step-workspace write

**Preconditions:** As J1. A stage whose agent attempts a write outside its assignment (under the mock LLM, a `step-executor.json` entry targeting a mismatched step id; with a real model, a prompt-injected step).

**Steps:**
1. Submit the job; watch the offending stage.

**Expected:**
- The before-tool-call guardrail refuses the `StepTools` write; it never lands.
- The step is recorded with the guardrail reason; the agent returns without a partial write. For a step write, the `ExecutorWorkflow` releases the claim and the step returns to `OPEN` for another executor.
- No content is written into a step or job the agent was not assigned, and no write lands on a job that is already `COMPLETED` or `FAILED`.

## J5 — Pause and resume of a running job

**Preconditions:** A job is in `EXECUTING` status (steps are on the board, executors are running).

**Steps:**
1. On the running job's card, click Pause (or `POST /api/jobs/{id}/pause`).
2. Watch the job and step boards.
3. After five seconds, click Resume (or `POST /api/jobs/{id}/resume`).

**Expected:**
- After pause: the job status changes to `PAUSED`; executor workflows stop claiming new steps. Any step already `CLAIMED` completes normally and the claim is marked `DONE`.
- The step board shows no new claims while the job is `PAUSED`.
- After resume: the job status returns to `EXECUTING`; executors resume polling the board. The job continues from the same execution point — no steps already `DONE` are re-run and the `retryCount` is not incremented.
- Attempting to pause a job that is not `EXECUTING` returns `409`.
- Attempting to resume a job that is not `PAUSED` returns `409`.

## J6 — Post-completion audit review

**Preconditions:** A job has reached `COMPLETED` (run J1 first).

**Steps:**
1. On the completed job's card, open the audit-review box.
2. Enter a reviewer id, pick `FLAGGED`, add a note, and Submit (or `POST /api/jobs/{id}/audit-review`).

**Expected:**
- The review is recorded against the job; the card shows the recorded `FLAGGED` outcome and notes.
- The job's status stays `COMPLETED` — the review does not change it.
- Posting an audit review to a job that is not yet `COMPLETED` returns `409` and records nothing.
