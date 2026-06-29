# User journeys

Acceptance journeys the generated system must pass.

## J1 — Propose an action

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/action-request` with `{ "taskDescription": "restart the cache layer on cluster-a" }`.
- **Expected:** response returns `{ actionId }`. Within ~30 s the action appears via SSE in `PROPOSED` with non-empty `proposalSummary`, `estimatedImpact`, and `actionPayload`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and execute

- **Preconditions:** an action in `PROPOSED` with a complete proposal (J1).
- **Steps:** `POST /api/actions/{actionId}/approve` with `{ "approvedBy": "ops-lead", "note": "looks good" }`.
- **Expected:** `ActionEntity` transitions to `APPROVED`; the workflow's next poll moves to `executeStep`; `ExecutionAgent` runs; within ~30 s status becomes `EXECUTED` with a non-null `outcome` shown in the UI.

## J3 — Reject a proposal

- **Preconditions:** an action in `PROPOSED` (J1).
- **Steps:** `POST /api/actions/{actionId}/reject` with `{ "rejectedBy": "ops-lead", "reason": "scope too broad, split into two tasks" }`.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown in the UI; the workflow ends; the execution step never runs.

## J4 — Execution guard blocks premature execution

- **Preconditions:** an action in `PROPOSED` (not yet approved).
- **Steps:** drive the workflow toward `executeStep` without an approval (e.g., the await-approval poll runs while status is still `PROPOSED`).
- **Expected:** the before-tool-call guardrail re-reads `ActionEntity.status`; because it is not `APPROVED`, the simulated execution tool does not run and no `outcome` is set. The action stays in `PROPOSED` until a human acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the Overview tab renders the sample description.

## J6 — Proposal completeness guard fires

- **Preconditions:** model provider returns an incomplete proposal (missing `actionPayload`).
- **Steps:** `POST /api/action-request` with a vague task description that produces an incomplete draft.
- **Expected:** the before-agent-response guardrail on `ProposalAgent` rejects the draft; `ProposalAgent` retries; the action either reaches `PROPOSED` with a complete proposal or the workflow surfaces a failure — the action never appears in `PROPOSED` with empty fields.
