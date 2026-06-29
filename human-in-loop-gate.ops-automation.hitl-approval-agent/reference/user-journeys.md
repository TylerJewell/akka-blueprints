# User journeys

Acceptance journeys the generated system must pass.

## J1 — Propose an action

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/action-request` with `{ "operationRequest": "scale web-frontend to 5 replicas" }`.
- **Expected:** response returns `{ actionId }`. Within ~30 s the action appears via SSE in `PROPOSED` with non-empty `actionType`, `target`, `rationale`, and `estimatedImpact`. The proposal completeness guardrail ensures all four fields are present. Approve and Deny controls appear on the App UI card.

## J2 — Approve and execute

- **Preconditions:** an action in `PROPOSED` with a complete proposal (J1).
- **Steps:** `POST /api/actions/{actionId}/approve` with an operator name and optional note.
- **Expected:** `ActionEntity` transitions to `APPROVED`; the workflow's next poll moves to `executeStep`; `ExecutionAgent` runs; within ~30 s status becomes `EXECUTED` with a non-null `outcome` and `executionDetails` shown in the UI card.

## J3 — Deny a proposal

- **Preconditions:** an action in `PROPOSED` (J1).
- **Steps:** `POST /api/actions/{actionId}/deny` with a reason.
- **Expected:** status becomes terminal `DENIED`; the deny reason is shown in the UI card; the workflow ends; the execution step never runs; no `outcome` or `executionDetails` are set.

## J4 — Execution guard blocks premature execution

- **Preconditions:** an action in `PROPOSED` (not yet approved).
- **Steps:** drive the workflow toward `executeStep` without an approval (for example, the await-approval poll fires while status is still `PROPOSED`).
- **Expected:** the before-tool-call guardrail on `ExecutionAgent` re-reads `ActionEntity.status`; because it is not `APPROVED`, the simulated execution tool does not run and no `outcome` is set. The action remains in `PROPOSED` until an operator acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit the Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills (`hitl` yellow, `guardrail` red); Risk Survey renders pre-filled answers with `TO_BE_COMPLETED_BY_DEPLOYER` values muted italic; the Overview README renders correctly.

## J6 — Concurrent proposals stay isolated

- **Preconditions:** service running; a model provider configured.
- **Steps:** submit two different operation requests in rapid succession, yielding `actionId-A` and `actionId-B`. Approve `actionId-A` and deny `actionId-B`.
- **Expected:** `actionId-A` reaches `EXECUTED` independently of `actionId-B`. `actionId-B` reaches `DENIED`. Neither workflow interferes with the other. Both actions appear in the actions list with correct statuses.
