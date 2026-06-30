# User journeys

Acceptance journeys the generated system must pass.

## J1 — Score a lead

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/score-request` with `{ "companyName": "Acme Corp", "contactEmail": "cto@acme.io" }`.
- **Expected:** response returns `{ leadId }`. Within ~30 s the lead appears via SSE in `SCORED` with non-null `score`, `scoreRationale`, and `scoreConfidence`. Approve/Reject controls show on the App UI card.

## J2 — Approve and qualify

- **Preconditions:** a lead in `SCORED` with a rationale (J1).
- **Steps:** `POST /api/leads/{leadId}/approve` with an approver name and optional note.
- **Expected:** `LeadEntity` transitions to `APPROVED`; the workflow's next poll moves to `qualifyStep`; `QualificationAgent` runs; within ~30 s status becomes `QUALIFIED` with non-null `qualificationVerdict` and `qualificationNextSteps` shown in the UI.

## J3 — Reject a scored lead

- **Preconditions:** a lead in `SCORED` (J1).
- **Steps:** `POST /api/leads/{leadId}/reject` with a reason.
- **Expected:** status becomes terminal `DISQUALIFIED`; the reason is shown; the workflow ends; the qualify step never runs.

## J4 — Qualify guard blocks premature qualification

- **Preconditions:** a lead in `SCORED` (not yet approved).
- **Steps:** drive the workflow toward `qualifyStep` without an approval (the await-review poll runs while status is still `SCORED`).
- **Expected:** the before-tool-call guardrail re-reads `LeadEntity.status`; because it is not `APPROVED`, the qualification tool does not run and no `qualificationVerdict` is set. The lead stays in `SCORED` until a human acts.

## J5 — Eval event captured

- **Preconditions:** a lead has been scored (J1).
- **Steps:** after scoring completes, check that an eval event was emitted (observable via the event log endpoint or Eval Matrix tab).
- **Expected:** one eval event exists for the lead, carrying the `leadId`, `score`, `confidence`, and input profile. The Eval Matrix tab shows E1 with its rationale and implementation notes.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, E1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders muted; the README/Overview renders without error.
