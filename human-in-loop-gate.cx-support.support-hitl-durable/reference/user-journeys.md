# User journeys

Acceptance journeys the generated system must pass.

## J1 — Triage a ticket

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/tickets` with `{ "subject": "...", "body": "..." }`.
- **Expected:** response returns `{ ticketId }`. Within ~30 s the ticket appears via SSE in `TRIAGED` with non-empty `triageSummary`, `triageCategory`, and `triagePriority`. Approve/Escalate buttons appear on the App UI card.

## J2 — Approve and resolve

- **Preconditions:** a ticket in `TRIAGED` with a triage summary (J1).
- **Steps:** `POST /api/tickets/{ticketId}/approve` with an operator name and note.
- **Expected:** `TicketEntity` transitions to `APPROVED`; the workflow's next poll moves to `resolveStep`; `ResolutionAgent` runs; within ~30 s status becomes `RESOLVED` with a non-null `resolution` shown in the UI.

## J3 — Escalate a ticket

- **Preconditions:** a ticket in `TRIAGED` (J1).
- **Steps:** `POST /api/tickets/{ticketId}/escalate` with an operator name and reason.
- **Expected:** status becomes terminal `ESCALATED`; the escalation reason is shown; the workflow ends; the resolution step never runs.

## J4 — Resolution guard blocks premature resolution

- **Preconditions:** a ticket in `TRIAGED` (not yet approved).
- **Steps:** drive the workflow toward `resolveStep` without an approval (e.g., the await-decision poll runs while status is still `TRIAGED`).
- **Expected:** the workflow self-schedules a 5-second resume timer; `resolveStep` is not entered; no `resolution` field is set; the ticket stays in `TRIAGED` until an operator acts.

## J5 — PII absent from persisted triage summary

- **Preconditions:** service running with a real or mock model provider.
- **Steps:** `POST /api/tickets` with a body containing a recognizable email address and phone number. Wait for `TRIAGED` status.
- **Expected:** `GET /api/tickets/{ticketId}` returns a `triageSummary` that does not contain the submitted email address or phone number. The raw body field is not persisted on the entity.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, S1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted italic; the Overview tab renders without errors.
