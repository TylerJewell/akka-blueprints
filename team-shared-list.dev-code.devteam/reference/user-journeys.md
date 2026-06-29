# User journeys — dev-team-task-board

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: project to completion

**Preconditions:** Service running on declared port (9714); a model-provider key set, or the mock LLM selected during scaffolding. The developer roster (`dev-1`, `dev-2`, `dev-3`) is running and idle-polling.

**Steps:**
1. Open `http://localhost:9714/`. The App UI tab is visible.
2. In the form, enter title "URL shortener" and a one-line description. Click Submit.
3. A `projectId` is returned; the project appears with status `PLANNING`.

**Expected:**
- Within a few seconds the team lead's plan lands: three to six task cards appear in the `Open` column, and the project moves to `READY` then `IN_PROGRESS`.
- Developers claim eligible tasks; each claimed card shows a developer id and moves through `Claimed` → `In progress` → `In review` → `Done`.
- A task with `dependsOn` stays in `Open` until its dependencies are `Done`.
- When every task is `Done`, the project moves to `COMPLETED`.
- The board updates live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two developers idle and at least one `OPEN` task with no unmet dependency.

**Steps:**
1. Submit a project whose plan has a single dependency-free task so multiple developers race for it.

**Expected:**
- Exactly one developer's id appears on the contested task; the task is claimed once.
- The other developers return to polling and pick up different tasks (or idle if none remain).
- No task is ever shown claimed by two developers; the claim count per task is one.

## J3 — Peer coordination

**Preconditions:** As J1. Under the mock LLM, the `developer.json` set includes an entry whose `peerRequest` targets `dev-2`; with a real model, a brief whose tasks force a cross-task dependency.

**Steps:**
1. Submit a project that produces a task a developer cannot finish without another developer's output.
2. Watch that task.

**Expected:**
- The task moves to `BLOCKED` with a reason naming the peer (e.g., "waiting on peer: dev-2").
- A `PeerMessage` appears in `dev-2`'s mailbox panel with the concrete question.
- `POST /api/mailbox/dev-2/messages/{id}/reply` (via the reply box) posts a reply; the blocked task returns to `OPEN` and is then picked up and completed.

## J4 — Destructive tool call refused

**Preconditions:** As J1. A task whose agent attempts a destructive workspace operation (under the mock LLM, an entry whose file path escapes the workspace or whose content triggers the destructive pattern; with a real model, a prompt-injected brief).

**Steps:**
1. Submit the project; watch the offending task.

**Expected:**
- The before-tool-call guardrail refuses the operation; it never executes.
- The task is recorded `BLOCKED` with the guardrail reason; the developer returns to polling.
- No file outside the task workspace is touched.

## J5 — Operator halt and resume

**Preconditions:** As J1, with at least one project in progress.

**Steps:**
1. Click Halt in the control strip (or `POST /api/control/halt` with a reason and actor).
2. Observe the team for ~10 s.
3. Click Resume (or `POST /api/control/resume`).

**Expected:**
- After Halt: no new tasks are claimed; developers idle. `GET /api/control` reports `halted: true` with the reason and actor; the App UI shows the red halted banner. Any tool call attempted while halted is refused.
- After Resume: `halted` returns to `false`; developers resume claiming and the in-progress project continues to completion.
