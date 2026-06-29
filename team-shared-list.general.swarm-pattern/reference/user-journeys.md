# User journeys — swarm-pattern

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: job to completion

**Preconditions:** Service running on declared port (9405); a model-provider key set, or the mock LLM selected during scaffolding. The worker roster (`worker-1`, `worker-2`, `worker-3`) is running and idle-polling.

**Steps:**
1. Open `http://localhost:9405/`. The App UI tab is visible.
2. In the form, enter title "Classify customer feedback" and a one-line description. Click Submit.
3. A `jobId` is returned; the job appears with status `PLANNING`.

**Expected:**
- Within a few seconds the coordinator's plan lands: three to five item cards appear in the `Open` column, and the job moves to `READY` then `IN_PROGRESS`.
- Workers claim eligible items; each claimed card shows a worker id and moves through `Claimed` → `In progress` → `In review` → `Done`.
- An item with `dependsOn` stays in `Open` until its dependencies are `Done`.
- When every item is `Done`, the job moves to `FINISHED`.
- The board updates live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two workers idle and at least one `OPEN` item with no unmet dependency.

**Steps:**
1. Submit a job whose plan has a single dependency-free item so multiple workers race for it.

**Expected:**
- Exactly one worker's id appears on the contested item; the item is claimed once.
- The other workers return to polling and pick up different items (or idle if none remain).
- No item is ever shown claimed by two workers; the claim count per item is one.

## J3 — Relay coordination

**Preconditions:** As J1. Under the mock LLM, the `worker.json` set includes an entry whose `relayRequest` targets `worker-2`; with a real model, a brief whose items force a cross-item dependency.

**Steps:**
1. Submit a job that produces an item a worker cannot finish without another worker's output.
2. Watch that item.

**Expected:**
- The item moves to `STALLED` with a reason naming the target worker (e.g., "waiting on relay: worker-2").
- A `RelayMessage` appears in `worker-2`'s mailbox panel with the concrete question.
- `POST /api/mailbox/worker-2/messages/{id}/reply` (via the reply box) posts a reply; the stalled item returns to `OPEN` and is then picked up and completed.

## J4 — Operator pause and resume

**Preconditions:** As J1, with at least one job in progress.

**Steps:**
1. Click Pause in the control strip (or `POST /api/control/pause` with a reason and actor).
2. Observe the swarm for ~10 s.
3. Click Resume (or `POST /api/control/resume`).

**Expected:**
- After Pause: no new items are claimed; workers idle. `GET /api/control` reports `paused: true` with the reason and actor; the App UI shows the red paused banner.
- After Resume: `paused` returns to `false`; workers resume claiming and the in-progress job continues to completion.
