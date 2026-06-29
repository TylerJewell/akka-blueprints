# User journeys

Acceptance journeys the generated system must pass.

## J1 — Plan a tool call

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/tool-requests` with `{ "goal": "..." }`.
- **Expected:** response returns `{ requestId }`. Within ~30 s the request appears via SSE in `PENDING_APPROVAL` with non-empty `toolName`, `parameters`, and `rationale`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and execute

- **Preconditions:** a request in `PENDING_APPROVAL` with a plan (J1).
- **Steps:** `POST /api/tool-requests/{requestId}/approve` with `{ "approvedBy": "...", "approverNote": "..." }`.
- **Expected:** `ToolRequestEntity` transitions to `APPROVED`; the workflow's next poll moves to `executeStep`; `ExecutorAgent` runs; within ~30 s status becomes `EXECUTED` with a non-null `executionOutput` shown in the UI.

## J3 — Edit parameters and approve

- **Preconditions:** a request in `PENDING_APPROVAL` (J1).
- **Steps:** `POST /api/tool-requests/{requestId}/approve` with `{ "approvedBy": "...", "approverNote": "...", "editedParameters": "{...}" }` where `editedParameters` differs from the original `parameters`.
- **Expected:** `ToolCallApproved` event records `editedParameters`; `executeStep` reads the entity and uses `editedParameters` (not the original plan); `executionOutput` reflects the edited parameter values; final status is `EXECUTED`.

## J4 — Reject a plan

- **Preconditions:** a request in `PENDING_APPROVAL` (J1).
- **Steps:** `POST /api/tool-requests/{requestId}/reject` with `{ "rejectedBy": "...", "reason": "..." }`.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown; the workflow ends; the execute step never runs.

## J5 — Execute guard blocks unapproved execution

- **Preconditions:** a request in `PENDING_APPROVAL` (not yet approved).
- **Steps:** drive the workflow toward `executeStep` without an approval (e.g., the await-approval poll runs while status is still `PENDING_APPROVAL`).
- **Expected:** the before-tool-call guardrail re-reads `ToolRequestEntity.status`; because it is not `APPROVED`, the simulated execute tool does not run and no `executionOutput` is set. The request stays in `PENDING_APPROVAL` until a human acts.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 2 controls (H1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the Overview tab renders the component inventory and API contract.
