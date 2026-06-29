# User journeys — hierarchical-workflow-automation

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: request to delivered operations report

**Preconditions:** Service running on the declared port (9811); a model-provider key set, or the mock LLM selected during scaffolding. The executor roster (`executor-1`, `executor-2`) is running and idle-polling the task board.

**Steps:**
1. Open `http://localhost:9811/`. The App UI tab is visible.
2. In the form, enter the description "Rotate TLS certificates across the API gateway cluster". Click Submit.
3. A `requestId` is returned; the request appears with status `SUBMITTED`.

**Expected:**
- The orchestrator plans a workflow; the request moves to `PLANNED` and the plan objective appears on the card.
- The discovery team runs: the lead plans scan targets, scanner instances each write a result into the workspace, and the lead synthesises a summary; the request moves through `DISCOVERING` to `DISCOVERED` and the discovery overview appears.
- The execution team runs: the lead plans tasks, task cards appear in the task board's `Open` group, and the request moves to `EXECUTING`. Executors claim tasks (each claimed card shows an executor id) and move them `Open` → `Claimed` → `Done`. The request-progress line tracks the count.
- When every task is `Done`, the execution summary assembles and the request moves to `EXECUTED`.
- The validation panel scores the result; with all axes passing the verdict is `PASS`, the request moves through `VALIDATING` to `VALIDATED`, the orchestrator assembles the report, and the request moves to `COMPLETED` with a generated URL.
- Three audit-eval score chips (discovery / execution / validation) appear on the card.
- Both boards update live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two executors idle and at least one `OPEN` task on the board.

**Steps:**
1. Submit a request so the execution team plans several tasks that land on the board at once.

**Expected:**
- Each task shows exactly one executor id; no task is ever shown claimed by two executors.
- Executors that lose a claim race return to polling and pick up different tasks (or idle if none remain).
- The claim count per task is one.

## J3 — Bounded retry loop

**Preconditions:** As J1. Under the mock LLM, the `validator.json` set includes at least one `RETRY` note naming a task title; with a real model, an execution summary with a task result that fails the `correctness` axis.

**Steps:**
1. Submit a request whose execution result draws a `RETRY` from one of the validation axes.
2. Watch the request.

**Expected:**
- The validation verdict is `RETRY`; the named task(s) reset to `OPEN` on the board and the request returns to `EXECUTING` with `retryCount` now 1.
- An executor claims and re-executes the reset task; the execution summary reassembles and the panel validates it again.
- On the second pass the request proceeds to report delivery (a second `RETRY` is accepted rather than looping again), so the pipeline terminates.

## J4 — Refused cross-system action

**Preconditions:** As J1. A stage whose agent attempts an action on a request or task it was not assigned (under the mock LLM, a `system-scanner.json` entry whose result targets a mismatched request id; with a real model, a prompt-injected description).

**Steps:**
1. Submit the request; watch the offending stage.

**Expected:**
- The before-tool-call guardrail refuses the `SystemTools` call; it never lands.
- The stage is recorded with the guardrail reason; the agent returns without a partial side effect. For a task action, the `ExecutorWorkflow` releases the claim and the task returns to `OPEN` for another executor.
- No result is written into a request or task the agent was not assigned, and no action lands on an already-completed request.

## J5 — Post-execution compliance review

**Preconditions:** A request has reached `COMPLETED` (run J1 first).

**Steps:**
1. On the completed request's card, open the compliance-review box.
2. Enter a reviewer id, pick `FLAGGED`, add a comment, and Submit (or `POST /api/requests/{id}/compliance-review`).

**Expected:**
- The review is recorded against the request; the card shows the recorded `FLAGGED` outcome and comment.
- The request's status stays `COMPLETED` — the review does not change it.
- Posting a compliance review to a request that is not yet `COMPLETED` returns `409` and records nothing.
