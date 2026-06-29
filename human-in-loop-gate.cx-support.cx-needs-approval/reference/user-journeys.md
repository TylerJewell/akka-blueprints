# User journeys

Acceptance journeys the generated system must pass.

## J1 — Triage a ticket

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/ticket-request` with `{ "customerMessage": "..." }`.
- **Expected:** response returns `{ ticketId }`. Within ~30 s the ticket appears via SSE in `TRIAGED` with non-empty `responseSummary` and `responseBody`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and resolve

- **Preconditions:** a ticket in `TRIAGED` with a response body (J1).
- **Steps:** `POST /api/tickets/{ticketId}/approve` with a supervisor name and optional note.
- **Expected:** `TicketEntity` transitions to `APPROVED`; the workflow's next poll moves to `resolveStep`; `ResolutionAgent` runs; within ~30 s status becomes `RESOLVED` with a non-null `confirmationId` shown in the UI.

## J3 — Reject a triage draft

- **Preconditions:** a ticket in `TRIAGED` (J1).
- **Steps:** `POST /api/tickets/{ticketId}/reject` with a reason.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown; the workflow ends; the resolution step never runs and no `confirmationId` is set.

## J4 — Resolution guard blocks premature delivery

- **Preconditions:** a ticket in `TRIAGED` (not yet approved).
- **Steps:** drive the workflow toward `resolveStep` without an approval (e.g., the await-approval poll runs while status is still `TRIAGED`).
- **Expected:** the before-tool-call guardrail re-reads `TicketEntity.status`; because it is not `APPROVED`, the simulated deliver-response tool does not run and no `confirmationId` is set. The ticket stays in `TRIAGED` until a supervisor acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the Overview tab renders without errors.

## J6 — SSE live update

- **Preconditions:** service running; UI open on App UI tab with an active SSE connection.
- **Steps:** `POST /api/ticket-request` from a second terminal while the UI is open.
- **Expected:** without page reload, the new ticket card appears in the tickets list as the SSE stream delivers the `TRIAGED` row from `TicketsView`. Approve/Reject buttons are present on the new card once `responseBody` is populated.
