# User journeys

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Plan an action from a request

- **Preconditions:** service running; one model-provider key resolved (or mock provider).
- **Steps:** POST `/api/requests` with `{ "request": "Email the team about the Friday outage" }`. Subscribe to `/api/actions/sse`.
- **Expected state:** within ~30 s the action transitions `CREATED → PLANNED` with non-empty `toolName` (one of the allowed tools), `toolArguments`, and `rationale`.
- **Expected UI:** the action appears in the App UI list in `PLANNED` state with Approve/Reject buttons.
- **Done when:** the streamed `Action` has `status == PLANNED` and `toolName` present.

## J2 — Approve and execute

- **Preconditions:** an action in `PLANNED` (J1).
- **Steps:** POST `/api/actions/{id}/approve` with `{ "approvedBy": "reviewer", "comment": "ok" }`.
- **Expected state:** status `PLANNED → APPROVED → EXECUTED`; the before-tool-call guardrail confirms `APPROVED`; `toolOutput` becomes non-empty and `executedAt` is set within ~30 s.
- **Expected UI:** the card shows the tool output; Approve/Reject buttons disappear.
- **Done when:** the streamed `Action` has `status == EXECUTED` with non-empty `toolOutput`.

## J3 — Reject an action

- **Preconditions:** an action in `PLANNED`.
- **Steps:** POST `/api/actions/{id}/reject` with `{ "rejectedBy": "reviewer", "reason": "out of scope" }`.
- **Expected state:** status `PLANNED → REJECTED` (terminal); `rejectReason` recorded; the tool never runs.
- **Expected UI:** the card shows the reject reason; no Approve/Reject buttons.
- **Done when:** the streamed `Action` has `status == REJECTED` with the reason set.

## J4 — Escalate a stale plan

- **Preconditions:** an action in `PLANNED`, left untouched.
- **Steps:** wait > 2 minutes without approving or rejecting.
- **Expected state:** `StuckActionMonitor` marks it `ESCALATED` (terminal); the tool never runs.
- **Expected UI:** the card shows `ESCALATED`; Approve/Reject buttons disappear.
- **Done when:** the streamed `Action` has `status == ESCALATED` with `escalatedAt` set.

## J5 — Background load from the simulator

- **Preconditions:** service running, no UI interaction.
- **Steps:** observe `/api/actions/sse`.
- **Expected state:** `RequestSimulator` drips a canned request every 30 s; `RequestConsumer` starts a workflow per request; new actions appear in `PLANNED`.
- **Done when:** at least one action appears without any manual POST.
