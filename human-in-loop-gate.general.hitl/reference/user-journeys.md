# User journeys

Acceptance journeys the generated system must pass.

## J1 — Propose an action

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/action-request` with `{ "requestText": "..." }`.
- **Expected:** response returns `{ actionId }`. Within ~30 s the action appears via SSE in `PROPOSED` with non-empty `proposalSummary`, `proposalRationale`, and a valid `proposalRiskLevel` (`LOW`, `MEDIUM`, or `HIGH`). Approve/Reject buttons show on the App UI card.

## J2 — Approve and execute

- **Preconditions:** an action in `PROPOSED` with a complete proposal (J1).
- **Steps:** `POST /api/actions/{actionId}/approve` with `{ "approvedBy": "reviewer-name", "comment": "looks good" }`.
- **Expected:** `ActionEntity` transitions to `APPROVED`; the workflow's next poll moves to `executeStep`; `ExecutionAgent` runs; within ~30 s status becomes `EXECUTED` with a non-null `executionOutcome` shown in the UI.

## J3 — Reject a proposal

- **Preconditions:** an action in `PROPOSED` (J1).
- **Steps:** `POST /api/actions/{actionId}/reject` with `{ "rejectedBy": "reviewer-name", "reason": "out of scope" }`.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown in the UI; the workflow ends; the execute step never runs.

## J4 — Execution guard blocks premature execution

- **Preconditions:** an action in `PROPOSED` (not yet approved).
- **Steps:** drive the workflow toward `executeStep` without an approval (e.g., the await-approval poll runs while status is still `PROPOSED`).
- **Expected:** the before-tool-call guardrail re-reads `ActionEntity.status`; because it is not `APPROVED`, the simulated execution tool does not run and no `executionOutcome` is set. The action stays in `PROPOSED` until a human acts.

## J5 — Proposal quality gate rejects malformed output

- **Preconditions:** service running with mock or real model provider.
- **Steps:** configure the mock to return a proposal with an empty `summary` or an invalid `riskLevel`.
- **Expected:** the before-agent-response guardrail intercepts the malformed proposal; the action does not transition to `PROPOSED`; the workflow step retries or surfaces an error without persisting bad data.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills (`hitl` yellow, `guardrail` red); Risk Survey renders pre-filled answers with deployer placeholders muted italic; the Overview tab renders the README content.
