# Architecture — self-correcting-extraction

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ExtractionEndpoint` is the entry point. It writes a `DocumentSubmitted` event to `DocumentQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `ExtractionWorkflow` per submission. The workflow alternates two agents — `ExtractionAgent` extracts structured fields, `ScorerAgent` grades them — with a memory-lookup step before scoring. Each step emits events on `ExtractionJobEntity`. `JobsView` projects those events into the read model the UI streams via SSE.

`MemoryEntity` sits alongside the main flow: the workflow writes confirmed field values to it on a `PASS` transition, and the Scorer reads from it at the start of each score step to ground its verdict in prior confirmed extractions. Two TimedActions complete the picture: `DocumentSimulator` drips a sample document every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any scored attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first extraction is returned for correction and the second extraction passes. The Extraction Agent → Scorer alternation is explicit, as is the memory read before each score step and the memory write after the final `PASS`. Each attempt's events are written to `ExtractionJobEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The job moves between two transient states (`EXTRACTING`, `SCORING`) and two terminal states (`VERIFIED`, `BUDGET_EXHAUSTED`). The `SCORING → EXTRACTING` transition is the `CORRECT` path — taken when the scorer returns `CORRECT` and the attempt count is still below the budget cap. `BUDGET_EXHAUSTED` is reached only when the ceiling is hit; both terminal states preserve every attempt and every correction note on the entity.

## Entity model

`ExtractionJobEntity` is the job source of truth; every transition writes one of six event types. `MemoryEntity` is the cross-job learning store; it grows one `MemoryRecordWritten` event per confirmed field per verified job. `DocumentQueue` is the audit log of raw submissions. `JobsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `extractStep` and `scoreStep`; memory reads are in-process against a local entity and effectively instant.
- Workflow-wide bound: the loop bounds itself with `budgetCap` (default 4). A wall-clock guard could be added by converting the cap to a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(exhaustStep)` — any unrecoverable agent failure ends in `BUDGET_EXHAUSTED`, not in a hung workflow.
- Window buffer: the workflow state carries the last three `FieldMap` results; each new extraction is prepended and the list is trimmed. The buffer is passed to `CORRECT_EXTRACTION` so the agent can detect persistent errors across attempts.
- Memory write ordering: `verifyStep` writes confirmed fields to `MemoryEntity` only after `JobVerified` is emitted on `ExtractionJobEntity`. The sequencing is enforced by workflow step order; no compensating transaction is needed.
- Idempotency: `ExtractionEndpoint.submit` deduplicates on `(documentType, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(jobId, attemptNumber)`.
