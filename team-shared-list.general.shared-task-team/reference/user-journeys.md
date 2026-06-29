# User journeys — shared-task-team

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: goal to completion

**Preconditions:** Service running on declared port (9881); a model-provider key set, or the mock LLM selected during scaffolding. The worker roster (`worker-1`, `worker-2`, `worker-3`) is running and idle-polling.

**Steps:**
1. Open `http://localhost:9881/`. The App UI tab is visible.
2. In the form, enter title "Cloud storage comparison" and a one-line description. Click Submit.
3. A `goalId` is returned; the goal appears with status `PLANNING`.

**Expected:**
- Within a few seconds the planner's plan lands: three to six task cards appear in the `Open` column, and the goal moves to `READY` then `IN_PROGRESS`.
- Workers claim eligible tasks; each claimed card shows a worker id and moves through `Claimed` → `In progress` → `In review` → `Done`.
- A task with `dependsOn` stays in `Open` until its dependencies are `Done`.
- When every task is `Done`, the goal moves to `COMPLETED` and the on-decision eval result is attached.
- The board updates live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two workers idle and at least one `OPEN` task with no unmet dependency.

**Steps:**
1. Submit a goal whose plan has a single dependency-free task so multiple workers race for it.

**Expected:**
- Exactly one worker's id appears on the contested task; the task is claimed once.
- The other workers return to polling and pick up different tasks (or idle if none remain).
- No task is ever shown claimed by two workers; the claim count per task is one.

## J3 — Peer coordination

**Preconditions:** As J1. Under the mock LLM, the `worker-agent.json` set includes an entry whose `peerRequest` targets `worker-2`; with a real model, a brief whose tasks force a cross-task dependency.

**Steps:**
1. Submit a goal that produces a task a worker cannot finish without another worker's output.
2. Watch that task.

**Expected:**
- The task moves to `BLOCKED` with a reason naming the peer (e.g., "waiting on peer: worker-2").
- A `PeerMessage` appears in `worker-2`'s mailbox panel with the concrete question.
- `POST /api/mailbox/worker-2/messages/{id}/reply` (via the reply box) posts a reply; the blocked task returns to `OPEN` and is then picked up and completed.

## J4 — Guardrail blocks unsafe synthesized output

**Preconditions:** As J1. A task whose worker agent produces a response that fails the before-agent-response guardrail (under the mock LLM, an entry whose section content triggers the refusal criteria; with a real model, a prompt that elicits a guardrail-triggering response).

**Steps:**
1. Submit the goal; watch the offending task.

**Expected:**
- The before-agent-response guardrail refuses the synthesized response; it is never written to the task artifact.
- The task is recorded `BLOCKED` with the guardrail reason; the worker returns to polling.
- No refused content is persisted anywhere in the system.

## J5 — Operator halt and resume

**Preconditions:** As J1, with at least one goal in progress.

**Steps:**
1. Click Halt in the control strip (or `POST /api/control/halt` with a reason and actor).
2. Observe the team for ~10 s.
3. Click Resume (or `POST /api/control/resume`).

**Expected:**
- After Halt: no new tasks are claimed; workers idle. `GET /api/control` reports `halted: true` with the reason and actor; the App UI shows the red halted banner. Any response generation attempted while halted is refused.
- After Resume: `halted` returns to `false`; workers resume claiming and the in-progress goal continues to completion.

## J6 — On-decision eval records consistency score

**Preconditions:** As J1, with a goal that completes fully.

**Steps:**
1. Submit a goal and wait for it to reach `COMPLETED`.
2. Call `GET /api/goals/{goalId}`.

**Expected:**
- The response body includes an `evalResult` field (non-null) with scores for coverage, absence of contradiction, and presence of a plan-level summary.
- The `status` field is `COMPLETED` regardless of the eval outcome — the eval is non-blocking.
