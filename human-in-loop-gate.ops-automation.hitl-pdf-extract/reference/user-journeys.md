# User journeys

Acceptance journeys the generated system must pass.

## J1 — Extract a document

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** `POST /api/extraction-request` with `{ "documentUrl": "https://example.com/invoice.pdf" }`.
- **Expected:** response returns `{ documentId }`. Within ~30 s the document appears via SSE in `EXTRACTED` with non-empty `rawFields` and `redactedFields` maps, and a `confidence` value between 0.0 and 1.0.

## J2 — High-confidence auto-approval and post

- **Preconditions:** service running; mock responses configured to return `confidence >= 0.85` (or real model returns high confidence).
- **Steps:** `POST /api/extraction-request` with a document URL.
- **Expected:** within ~30 s status transitions automatically through `EXTRACTED` → `APPROVED` → `POSTED` without any human action. `postTarget` is non-null in the final state. No Approve/Reject buttons appear in the UI because the document never enters `PENDING_REVIEW`.

## J3 — Low-confidence review and approve

- **Preconditions:** service running; mock responses configured to return `confidence < 0.85` (or real model returns low confidence); a document in `PENDING_REVIEW` (J1).
- **Steps:** `POST /api/documents/{documentId}/approve` with `{ "reviewedBy": "ops-team", "comment": "fields look correct" }`.
- **Expected:** `DocumentEntity` transitions to `APPROVED`; the workflow's next poll moves to `postStep`; data is posted downstream; within ~30 s status becomes `POSTED` with a non-null `postTarget` shown in the UI.

## J4 — Reject a low-confidence extraction

- **Preconditions:** a document in `PENDING_REVIEW` (J1).
- **Steps:** `POST /api/documents/{documentId}/reject` with `{ "rejectedBy": "ops-team", "reason": "vendor name not legible" }`.
- **Expected:** status becomes terminal `REJECTED`; the reason is displayed in the UI; the workflow ends; the post step never runs.

## J5 — Post guard blocks unapproved posting

- **Preconditions:** a document in `PENDING_REVIEW` (not yet approved).
- **Steps:** drive the workflow toward `postStep` without an approval (e.g., the await-review poll runs while status is still `PENDING_REVIEW`).
- **Expected:** the before-tool-call guardrail re-reads `DocumentEntity.status`; because it is not `APPROVED`, the simulated post tool does not run and no `postTarget` is set. The document remains in `PENDING_REVIEW` until a human acts.

## J6 — Metadata tabs render

- **Preconditions:** service running.
- **Steps:** open the UI; visit Risk Survey and Eval Matrix tabs.
- **Expected:** Eval Matrix renders 3 controls (H1, S1, G1) in `matrix-row` style with mechanism pills; Risk Survey renders pre-filled answers with deployer placeholders rendered muted italic; the Overview/README renders.
