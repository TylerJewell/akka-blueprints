# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a 4-task graph in the human-in-loop-gate pattern: extract, redact, await review (conditionally), post downstream.

## Component graph

`ExtractionEndpoint` accepts a document URL and starts `ExtractionWorkflow` with a fresh document id. The workflow drives `ExtractionAgent` to extract structured fields, writes the raw result to `DocumentEntity`, then drives `RedactionAgent` to produce a PII-scrubbed copy. After redaction, the workflow reads the confidence score: if it meets the threshold the document is auto-approved and moves directly to `postStep`; if not, `awaitReviewStep` pauses at a `PENDING_REVIEW` gate. A human calls approve or reject through the endpoint, which transitions `DocumentEntity`. On approval the workflow executes the simulated post and records the receipt. `DocumentEntity` events project into `DocumentsView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary low-confidence journey runs: submit → extract → redact → pause → human approve → post. The await-review task does not hold a thread: `awaitReviewStep` reads `DocumentEntity.getDocument`, and while the status is `PENDING_REVIEW` it self-schedules a 5-second resume timer. When the human transitions the status to `APPROVED`, the next poll moves the workflow to `postStep`. A `REJECTED` status ends the workflow. For high-confidence documents the workflow skips the pause entirely, calling `approve` directly on `DocumentEntity` before proceeding to `postStep`.

## State machine

A document moves `EXTRACTED → APPROVED → POSTED` on the high-confidence path. On the low-confidence path it passes through `PENDING_REVIEW` before reaching `APPROVED`. The terminal rejection path is `PENDING_REVIEW → REJECTED`. Each transition is driven by a corresponding event (`DocumentApproved`, `DocumentPosted`, `DocumentRejected`). The `PENDING_REVIEW` state is only reachable from `EXTRACTED`, never re-entered.

## Entity model

`DocumentEntity` is event-sourced; each command emits one event that the applier folds into the `Document` state. Both `rawFields` and `redactedFields` are persisted on the entity so the full audit trail is intact while the UI exposes only the sanitized version. `DocumentsView` projects the same events into a row keyed by document id. There is one view query, `getAllDocuments`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
