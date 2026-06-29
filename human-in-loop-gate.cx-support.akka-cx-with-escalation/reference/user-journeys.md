# User journeys

Acceptance journeys the generated system must pass.

## J1 — Open a conversation and receive a reply

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/conversations` with `{ "customerMessage": "My order has not arrived after two weeks." }`.
- **Expected:** response returns `{ conversationId }`. Within ~30 s the conversation appears via SSE in `ACTIVE` with non-empty `agentReply`. The App UI card shows Escalate and Resolve buttons.

## J2 — Escalate a conversation and accept it

- **Preconditions:** a conversation in `ACTIVE` (J1).
- **Steps:** `POST /api/conversations/{conversationId}/escalate` with `{ "reason": "Customer requested human agent" }`. Then `POST /api/conversations/{conversationId}/accept-escalation` with `{ "acceptedBy": "agent-alice" }`.
- **Expected:** after the escalate call, status moves to `ESCALATING`. After the accept call, `acceptedBy` is set and the workflow's next poll transitions to `acceptEscalationStep`. Within ~30 s status becomes `ESCALATED` with non-null `handoffSummary` and a `handoffPriority` value shown in the UI.

## J3 — Resolve a conversation directly

- **Preconditions:** a conversation in `ACTIVE` (J1).
- **Steps:** `POST /api/conversations/{conversationId}/resolve` with `{ "note": "Issue confirmed resolved." }`.
- **Expected:** status becomes terminal `RESOLVED`; the resolve note is shown; the workflow ends; the escalation path is never taken and `handoffSummary` remains null.

## J4 — Escalation gate blocks handoff without human acceptance

- **Preconditions:** a conversation in `ESCALATING` (first step of J2 completed; accept step not called yet).
- **Steps:** observe the `awaitEscalationDecisionStep` polling cycle while the conversation is in `ESCALATING` but `acceptedBy` is absent.
- **Expected:** `acceptEscalationStep` does not call `EscalationAgent` until `acceptedBy` is set. No `handoffSummary` is recorded. The conversation stays in `ESCALATING` until a human accepts.

## J5 — PII is not stored on the entity

- **Preconditions:** service running.
- **Steps:** `POST /api/conversations` with `{ "customerMessage": "My email is customer@example.com and my card 4111111111111111 was charged twice." }`. Wait for `ACTIVE`. Then `GET /api/conversations/{conversationId}`.
- **Expected:** `customerMessage` on the returned entity contains redacted placeholders (e.g. `[EMAIL]`, `[CARD]`) in place of the original personal data. The `agentReply` does not reproduce the raw values.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, S1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted italic; the Overview tab loads the README content.
