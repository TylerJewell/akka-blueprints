# User journeys

Acceptance journeys the generated system must pass.

## J1 — Generate a plan

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/plan-request` with `{ "goal": "..." }`.
- **Expected:** response returns `{ planId }`. Within ~30 s the plan appears via SSE in `PLANNED` with a non-empty `steps` list (at least two items) and a non-empty `rationale`. Edit, Approve, and Cancel controls appear on the App UI card.

## J2 — Edit a plan before approval

- **Preconditions:** a plan in `PLANNED` with steps (J1).
- **Steps:** `PATCH /api/plans/{planId}/edit` with `{ "editedBy": "...", "revisedSteps": ["...", "..."] }`.
- **Expected:** `PlanEntity` records a `PlanEdited` event; status remains `PLANNED`; the updated steps appear in the UI within ~5 s via SSE. The workflow does not advance.

## J3 — Approve and execute

- **Preconditions:** a plan in `PLANNED` (J1 or J2).
- **Steps:** `POST /api/plans/{planId}/approve` with an approver name and optional note.
- **Expected:** `PlanEntity` transitions to `APPROVED`; the workflow's next poll moves to `executeStep`; `ExecutionAgent` runs; within ~30 s status becomes `EXECUTED` with a non-null `outcome` shown in the UI.

## J4 — Cancel a plan

- **Preconditions:** a plan in `PLANNED` (J1).
- **Steps:** `POST /api/plans/{planId}/cancel` with a reason.
- **Expected:** status becomes terminal `CANCELLED`; the reason is shown; the workflow ends; the execute step never runs.

## J5 — Execution guard blocks premature execution

- **Preconditions:** a plan in `PLANNED` (not yet approved).
- **Steps:** drive the workflow toward `executeStep` without an approval (e.g., the await-approval poll runs while status is still `PLANNED`).
- **Expected:** the before-tool-call guardrail re-reads `PlanEntity.status`; because it is not `APPROVED`, the execute tool does not run and no `outcome` is written. The plan stays in `PLANNED` until a human acts.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted italic; Overview renders the README content.
