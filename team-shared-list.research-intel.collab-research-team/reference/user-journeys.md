# User journeys — collab-research-team

Acceptance criteria. The generated system passes when all six journeys complete as written.

## J1 — Happy path: question to completed report

**Preconditions:** Service running on declared port (9755); a model-provider key set, or the mock LLM selected during scaffolding. The researcher roster (`researcher-1`, `researcher-2`, `researcher-3`) is running and idle-polling.

**Steps:**
1. Open `http://localhost:9755/`. The App UI tab is visible.
2. In the form, enter the question "What are the leading causes of urban heat islands?" with a one-line context note. Click Submit.
3. A `questionId` is returned; the question appears with status `PLANNING`.

**Expected:**
- Within a few seconds the coordinator's plan lands: three to five sub-topic task cards appear in the `Open` column, and the question moves to `READY` then `IN_PROGRESS`.
- Researchers claim sub-topics; each claimed card shows a researcher id and moves through `Claimed` → `In progress` → `Findings Ready` → `Done`.
- When every sub-topic is `Done`, synthesis starts automatically; the question moves to `SYNTHESIZING`.
- If every conclusion is citation-grounded, the question moves to `COMPLETED` and the report panel shows the executive summary and conclusions.
- The board updates live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two researchers idle and at least one `OPEN` sub-topic task.

**Steps:**
1. Submit a question whose plan has a single sub-topic so multiple researchers race for it.

**Expected:**
- Exactly one researcher's id appears on the contested task; the task is claimed once.
- The other researchers return to polling and pick up different tasks (or idle if none remain).
- No task is ever shown claimed by two researchers; the claim count per task is one.

## J3 — Researcher coordination

**Preconditions:** As J1. Under the mock LLM, the `researcher.json` set includes an entry whose `coordinationRequest` targets `researcher-2`; with a real model, a question whose sub-topics have a real scope dependency.

**Steps:**
1. Submit a question that produces a sub-topic a researcher cannot scope without another researcher's output.
2. Watch that task.

**Expected:**
- The task moves to `BLOCKED` with a reason naming the peer (e.g., "waiting on: researcher-2").
- A `CoordinationMessage` appears in `researcher-2`'s mailbox panel with the concrete question.
- `POST /api/mailbox/researcher-2/messages/{id}/reply` (via the reply box) posts a reply; the blocked task returns to `OPEN` and is then claimed and completed.

## J4 — Citation-grounding eval: question flagged for review

**Preconditions:** As J1. Under the mock LLM, the `synthesis-agent.json` set includes an entry whose first conclusion has an empty `citedUrls` list.

**Steps:**
1. Submit a question; wait for synthesis to complete.

**Expected:**
- The citation-grounding eval (E1) fires per conclusion; the un-grounded conclusion is detected.
- The question moves to `NEEDS_REVIEW` instead of `COMPLETED`.
- The report panel shows the un-grounded conclusion highlighted and an Approve / Reject control.
- `POST /api/questions/{id}/approve` moves the question to `COMPLETED`; the report is then visible in full.

## J5 — Minimum-source guardrail: researcher report blocked

**Preconditions:** As J1. Under the mock LLM, the `researcher.json` set includes an entry with only one source URL (triggering the G1 guardrail minimum-source check).

**Steps:**
1. Submit a question; watch the sub-topic whose mock response has a single source.

**Expected:**
- The after-llm-response guardrail (G1) fires; the insufficient `FindingsReport` is rejected before it reaches `FindingsSubmitted` on the entity.
- The task is recorded `BLOCKED` with the guardrail reason (e.g., "findings report has fewer than 2 distinct sources").
- The researcher returns to polling; the blocked task is eventually released by `StuckTaskMonitor` and picked up.

## J6 — Operator halt and resume

**Preconditions:** As J1, with at least one question in progress.

**Steps:**
1. Click Halt in the control strip (or `POST /api/control/halt` with a reason and actor).
2. Observe the team for ~10 s.
3. Click Resume (or `POST /api/control/resume`).

**Expected:**
- After Halt: no new tasks are claimed; researchers idle. `GET /api/control` reports `halted: true` with the reason and actor; the App UI shows the red halted banner. Any in-flight guardrail check is refused.
- After Resume: `halted` returns to `false`; researchers resume claiming and the in-progress question continues through synthesis to completion.
