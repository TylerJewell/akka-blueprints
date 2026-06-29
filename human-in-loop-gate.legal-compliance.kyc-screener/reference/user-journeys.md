# User journeys

Acceptance journeys the generated system must pass.

## J1 — Screen an entity

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/screening-request` with `{ "entityId": "E-001", "documents": ["[DOC:1] Entity has no adverse media entries."] }`.
- **Expected:** response returns `{ caseId }`. Within ~30 s the case appears via SSE in `SCREENED` with non-empty `findings` that include at least one citation (e.g., `[DOC:1]`). Approve/Reject buttons show on the App UI card.

## J2 — Approve and close

- **Preconditions:** a case in `SCREENED` with findings (J1).
- **Steps:** `POST /api/cases/{caseId}/approve` with an officer name and note.
- **Expected:** `CaseEntity` transitions to `OFFICER_APPROVED`; the workflow's next poll moves to `closeStep`; `VerdictAgent` runs; within ~30 s status becomes `CLOSED` with `disposition` `APPROVED` shown in the UI.

## J3 — Reject a screening result

- **Preconditions:** a case in `SCREENED` (J1).
- **Steps:** `POST /api/cases/{caseId}/reject` with a reason.
- **Expected:** status becomes terminal `REJECTED`; the reason is shown; the workflow ends; the close step never runs.

## J4 — Close guard blocks unapproved close

- **Preconditions:** a case in `SCREENED` (not yet officer-approved).
- **Steps:** drive the workflow toward `closeStep` without an officer approval (e.g., the await-approval poll runs while status is still `SCREENED`).
- **Expected:** the workflow remains in `awaitApprovalStep`; `VerdictAgent` is not called; no `closedAt` or `disposition` is set. The case stays in `SCREENED` until an officer acts.

## J5 — Citation guardrail rejects uncited findings

- **Preconditions:** mock provider configured to return a `ScreeningResult` with no citation tokens in `findings`.
- **Steps:** `POST /api/screening-request` and wait for `screenStep` to run.
- **Expected:** the before-agent-response guardrail on `ScreenerAgent` rejects the uncited result; the agent retries; if retries are exhausted the step fails with a clear error; no `SCREENED` case appears.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (S1, H1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the README/Overview renders.
