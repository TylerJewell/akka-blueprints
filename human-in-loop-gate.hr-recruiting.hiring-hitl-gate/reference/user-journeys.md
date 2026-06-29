# User journeys

Acceptance journeys the generated system must pass.

## J1 — Propose on a candidate application

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/hiring-request` with `{ "candidateName": "...", "role": "...", "applicationSummary": "..." }`.
- **Expected:** response returns `{ applicationId }`. Within ~30 s the application appears via SSE in `PROPOSED` with non-empty `hiringRecommendation`, `hiringRationale`, `meetingSubject`, and `meetingBody`. Approve/Decline buttons show on the App UI card.

## J2 — Approve and record outcome

- **Preconditions:** an application in `PROPOSED` with proposals present (J1).
- **Steps:** `POST /api/applications/{applicationId}/approve` with an approver name and optional note.
- **Expected:** `ApplicationEntity` transitions to `APPROVED`; the workflow's next poll moves to `recordOutcomeStep`; `DecisionsReachedService` runs; within ~30 s status becomes `DECIDED` with a non-null `decidedAt` shown in the UI.

## J3 — Decline a proposal

- **Preconditions:** an application in `PROPOSED` (J1).
- **Steps:** `POST /api/applications/{applicationId}/decline` with a reason.
- **Expected:** status becomes terminal `DECLINED`; the reason is shown in the UI; the workflow ends; the outcome-recording step never runs.

## J4 — Outcome guard blocks premature recording

- **Preconditions:** an application in `PROPOSED` (not yet approved).
- **Steps:** drive the workflow toward `recordOutcomeStep` without an approval (e.g., the await-validator poll runs while status is still `PROPOSED`).
- **Expected:** the before-tool-call guardrail re-reads `ApplicationEntity.status`; because it is not `APPROVED`, the outcome-recording tool does not run and no `decidedAt` is set. The application stays in `PROPOSED` until a human acts.

## J5 — Protected-attribute content is removed from proposals

- **Preconditions:** service running with a real LLM provider.
- **Steps:** `POST /api/hiring-request` with an `applicationSummary` that contains age, gender, or ethnicity signals.
- **Expected:** the `HiringProposal` stored on `ApplicationEntity` does not contain those signals. The sanitizer has removed or neutralised the language before the proposal was persisted.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, S1, G1) in `matrix-row` style with mechanism pills (`hitl` yellow, `sanitizer` blue, `guardrail` red); Risk Survey renders pre-filled answers with deployer placeholders muted italic; the Overview tab renders the component list.
