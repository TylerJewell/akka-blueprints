# User journeys

Acceptance journeys the generated system must pass.

## J1 — Submit a job and see analysis

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/jobs` with `{ "requestPayload": "..." }`.
- **Expected:** response returns `{ jobId }`. Within ~30 s the job appears via SSE in `ANALYZED` with non-empty `findings` and a `riskLevel` of `LOW`, `MEDIUM`, or `HIGH`. Approve/Reject buttons show on the App UI card.

## J2 — Approve and complete

- **Preconditions:** a job in `ANALYZED` with findings (J1).
- **Steps:** `POST /api/jobs/{jobId}/approve` with an approver name.
- **Expected:** `JobEntity` transitions to `APPROVED`; the workflow's next poll moves to `completeStep`; `CompletionAgent` runs; within ~30 s status becomes `COMPLETED` with a non-null `output` shown in the UI.

## J3 — Reject a job

- **Preconditions:** a job in `ANALYZED` (J1).
- **Steps:** `POST /api/jobs/{jobId}/reject` with a reason.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown in the UI; the workflow ends; the completion step never runs.

## J4 — Completion guard blocks premature completion

- **Preconditions:** a job in `ANALYZED` (not yet approved).
- **Steps:** drive the workflow toward `completeStep` without an approval (e.g., the await-approval poll runs while status is still `ANALYZED`).
- **Expected:** the before-tool-call guardrail re-reads `JobEntity.status`; because it is not `APPROVED`, the completion tool does not run and no `output` is set. The job stays in `ANALYZED` until a human acts.

## J5 — SSE stream reflects all transitions

- **Preconditions:** service running; a browser tab open on the App UI.
- **Steps:** submit a job (J1), then approve it (J2) without refreshing the page.
- **Expected:** the job card updates in place from the initial submitted state through `ANALYZED` to `APPROVED` to `COMPLETED` without any manual page refresh. Each transition arrives via the SSE stream.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 2 controls (H1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted italic; the Overview tab shows the Try it card with `/akka:build`.
