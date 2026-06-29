# User journeys

Acceptance journeys the generated system must pass.

## J1 — Plan a request

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/confirmation-request` with `{ "description": "Set up a weekly team report and share it with the engineering channel" }`.
- **Expected:** response returns `{ requestId }`. Within ~30 s the request appears via SSE in `PENDING_CONFIRMATION` with a non-empty `proposedActions` list (at least 2 items). Confirm/Cancel buttons show on the App UI card.

## J2 — Confirm and execute

- **Preconditions:** a request in `PENDING_CONFIRMATION` with proposed actions (J1).
- **Steps:** `POST /api/requests/{requestId}/confirm` with a confirmer name and optional note.
- **Expected:** `RequestEntity` transitions to `CONFIRMED`; the workflow's next poll moves to `executeStep`; `ExecutorAgent` runs; within ~30 s status becomes `EXECUTED` with a non-null `executionSummary` shown in the UI.

## J3 — Cancel a request

- **Preconditions:** a request in `PENDING_CONFIRMATION` (J1).
- **Steps:** `POST /api/requests/{requestId}/cancel` with a reason.
- **Expected:** status becomes terminal `CANCELLED`; the reason is shown; the workflow ends; the execute step never runs.

## J4 — Execution guard blocks unconfirmed execution

- **Preconditions:** a request in `PENDING_CONFIRMATION` (not yet confirmed).
- **Steps:** drive the workflow toward `executeStep` without a confirmation (e.g., the await-confirmation poll runs while status is still `PENDING_CONFIRMATION`).
- **Expected:** the before-tool-call guardrail re-reads `RequestEntity.status`; because it is not `CONFIRMED`, the simulated execution tool does not run and no `executionSummary` is set. The request stays in `PENDING_CONFIRMATION` until a human acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 2 controls (H1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the Overview tab renders without errors.

## J6 — SSE stream delivers live updates

- **Preconditions:** service running; App UI open in browser.
- **Steps:** submit a request via the App UI input; observe the requests list without reloading the page.
- **Expected:** the new request card appears in real time as SSE pushes `RequestsView` updates; status badge updates from `PENDING_CONFIRMATION` to `EXECUTED` without a page reload after confirmation.
