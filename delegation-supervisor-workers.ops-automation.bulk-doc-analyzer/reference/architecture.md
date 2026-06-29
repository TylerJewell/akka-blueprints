# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A document batch enters through `DocumentEndpoint` (`POST /api/documents/batches`) or is dripped by `DocumentSimulator` every 90 seconds. Either path writes a `BatchSubmitted` event onto `BatchQueue`. `BatchRequestConsumer` subscribes to those events, creates a `BatchEntity`, and starts one `BatchWorkflow` per document in the batch, each keyed by `documentId`.

The workflow is the supervisor for each document. It asks `BatchCoordinator` to partition the document into an extraction instruction and a classification instruction, fans the work out to `Extractor` and `Classifier` in parallel, then asks `BatchCoordinator` to merge the outputs and run PII sanitization inline. Every transition is written as a command to `DocumentEntity` (for per-document state) and `BatchEntity` (for aggregate batch status); their events project into `DocumentView`. The endpoint reads and streams the view. `QualitySampler` reads the view to pick a finished document to score.

## Interaction sequence

The sequence diagram traces the primary journey for one document within a batch: partition, parallel fan-out (extract + classify), join, merge with PII sanitization, and persistence. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. `BatchEntity` receives a `documentDone` command after each document finishes, and emits `BatchComplete` or `BatchPartiallyComplete` when the last document in the batch is resolved.

## State machine

`Document` moves `QUEUED → PROCESSING`, then to one of two terminals: `PROCESSED` (merge + sanitization succeeded) or `DEGRADED` (a worker timed out and the merge ran on partial input). `PROCESSED` accepts one further `QualityScored` self-transition when `QualitySampler` records a score.

`Batch` follows a parallel lifecycle: `PENDING → IN_PROGRESS`, then to `COMPLETE` (all documents processed) or `PARTIAL` (at least one document degraded).

## Entity model

`BatchQueue` seeds one `Batch` per submission and one `Document` per item in the batch. A document owns at most one `ExtractedFields`, one `DocumentClassification`, and one `ProcessedDocument`. The view row mirrors the document with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). The view also carries the parent batch's status so the App UI can show batch-level progress without a separate request.
