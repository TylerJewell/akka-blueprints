# User journeys

Acceptance journeys the generated system must pass.

## J1 — Triage a customer request

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/cases` with `{ "customerRequest": "My account was charged twice for the same order." }`.
- **Expected:** response returns `{ caseId }`. Within ~30 s the case appears via SSE in `TRIAGED` with non-empty `triageSummary`, `proposedResolution`, and a valid `urgency` value (`LOW`, `MEDIUM`, or `HIGH`). Approve/Decline buttons show on the App UI card.

## J2 — Approve and fulfil

- **Preconditions:** a case in `TRIAGED` with a proposed resolution (J1).
- **Steps:** `POST /api/cases/{caseId}/approve` with `{ "approvedBy": "specialist@example.com", "comment": "Confirmed duplicate charge." }`.
- **Expected:** `CaseEntity` transitions to `APPROVED`; the workflow's next poll moves to `fulfilStep`; `FulfillmentAgent` runs; within ~30 s status becomes `FULFILLED` with a non-null `confirmationId` shown in the UI.

## J3 — Decline a triage result

- **Preconditions:** a case in `TRIAGED` (J1).
- **Steps:** `POST /api/cases/{caseId}/decline` with `{ "declinedBy": "specialist@example.com", "reason": "Insufficient evidence of duplicate charge." }`.
- **Expected:** status becomes terminal `DECLINED`; the decline reason is shown in the UI; the workflow ends; the fulfilment step never runs and no `confirmationId` is set.

## J4 — Fulfilment guard blocks premature fulfilment

- **Preconditions:** a case in `TRIAGED` (not yet approved).
- **Steps:** drive the workflow toward `fulfilStep` without an approval (e.g., the await-approval poll runs while status is still `TRIAGED`).
- **Expected:** the before-tool-call guardrail re-reads `CaseEntity.status`; because it is not `APPROVED`, the simulated fulfilment tool does not run and no `confirmationId` is set. The case stays in `TRIAGED` until a specialist acts.

## J5 — High-urgency case routing

- **Preconditions:** service running; a model provider configured.
- **Steps:** `POST /api/cases` with a request that describes a service outage or safety concern.
- **Expected:** `TriageResult.urgency` is `HIGH`; the urgency badge renders red in the App UI; the case card is otherwise identical in structure to lower-urgency cases.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the Overview tab renders without errors.
