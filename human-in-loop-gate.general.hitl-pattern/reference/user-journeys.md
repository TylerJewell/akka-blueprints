# User journeys

Acceptance journeys the generated system must pass.

## J1 — Process a task

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/task-request` with `{ "request": "..." }`.
- **Expected:** response returns `{ taskId }`. Within ~30 s the task appears via SSE in `PROCESSED` with non-empty `taskSummary` and `taskRecommendation`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and finalize

- **Preconditions:** a task in `PROCESSED` with a recommendation (J1).
- **Steps:** `POST /api/tasks/{taskId}/approve` with an approver name.
- **Expected:** `TaskEntity` transitions to `APPROVED`; the workflow's next poll moves to `finalizeStep`; `TaskFinalizerAgent` runs; within ~30 s status becomes `FINALIZED` with a non-null `finalizedOutcome` shown in the UI.

## J3 — Reject a recommendation

- **Preconditions:** a task in `PROCESSED` (J1).
- **Steps:** `POST /api/tasks/{taskId}/reject` with a reason.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown; the workflow ends; the finalize step never runs.

## J4 — Finalize guard blocks premature finalization

- **Preconditions:** a task in `PROCESSED` (not yet approved).
- **Steps:** drive the workflow toward `finalizeStep` without an approval (e.g., the await-approval poll runs while status is still `PROCESSED`).
- **Expected:** the before-tool-call guardrail re-reads `TaskEntity.status`; because it is not `APPROVED`, the simulated finalize tool does not run and no `finalizedOutcome` is set. The task stays in `PROCESSED` until a human acts.

## J5 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the Overview tab renders correctly.

## J6 — SSE stream updates on approval

- **Preconditions:** a task in `PROCESSED` visible in the App UI (J1); App UI tab is open with an active SSE connection.
- **Steps:** approve the task via `POST /api/tasks/{taskId}/approve`.
- **Expected:** the SSE stream delivers a `task` event within seconds; the UI card's status badge updates from `PROCESSED` to `APPROVED` without a page reload. After `finalizeStep` completes, a second event updates the badge to `FINALIZED`.
