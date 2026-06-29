# User journeys

Acceptance journeys the generated system must pass.

## J1 — Submit a query and observe planning

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/research-request` with `{ "query": "What are the current regulatory approaches to AI transparency in the EU and US?" }`.
- **Expected:** response returns `{ reportId }`. Within ~15 s the report appears via SSE in `PLANNING` then advances to `INVESTIGATING` with a non-empty `subTopics` list of 3–5 items. The App UI card shows each sub-topic string.

## J2 — Await synthesis completion

- **Preconditions:** a report in `INVESTIGATING` (J1).
- **Steps:** wait for the workflow to complete all investigation tasks and run SynthesisAgent.
- **Expected:** within ~60–120 s the report status advances through `SYNTHESISING` to `AWAITING_REVIEW`. `reportTitle` is a non-empty string; `reportBody` has at least 4 paragraphs of text; `sourcesUsed` contains at least one URL. The App UI shows Approve and Reject buttons on the card.

## J3 — Approve and deliver

- **Preconditions:** a report in `AWAITING_REVIEW` with a non-empty body (J2).
- **Steps:** `POST /api/reports/{reportId}/approve` with `{ "reviewedBy": "analyst@example.com", "notes": "Looks good, approved for delivery." }`.
- **Expected:** status transitions to `APPROVED` then within ~5 s to `DELIVERED`. `deliveredAt` is a non-null ISO-8601 timestamp shown in the UI. The deliver step never executes without a prior approval.

## J4 — Reject and revise

- **Preconditions:** a report in `AWAITING_REVIEW` (J2).
- **Steps:** `POST /api/reports/{reportId}/reject` with `{ "reviewedBy": "analyst@example.com", "notes": "The EU section lacks citations; please add primary sources." }`.
- **Expected:** status transitions to `NEEDS_REVISION`. Within ~60 s the workflow re-enters `SYNTHESISING` with the revision notes injected. The report returns to `AWAITING_REVIEW` with an updated `reportBody` that addresses the reviewer's notes. Approve and Reject buttons reappear.

## J5 — Deliver guard blocks premature delivery

- **Preconditions:** a report in `AWAITING_REVIEW` (not yet approved).
- **Steps:** drive the workflow toward the deliver step without an approval (e.g., the awaitReviewStep poll runs while status is still `AWAITING_REVIEW`).
- **Expected:** the before-tool-call guardrail re-reads `ResearchEntity.status`; because it is not `APPROVED`, the simulated deliver tool does not run and `deliveredAt` remains null. The report stays in `AWAITING_REVIEW` until a human acts.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, G1, G2) in `matrix-row` style with mechanism pills (`hitl` yellow, `guardrail` red); Risk Survey renders pre-filled answers with deployer placeholders rendered muted italic; the Overview tab shows the component list and Try-it card with `/akka:build`.
