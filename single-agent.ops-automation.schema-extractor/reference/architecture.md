# Architecture — schema-extractor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ExtractionEndpoint` accepts a submission and writes a `DocumentSubmitted` event onto `ExtractionJobEntity`. The `DocumentSanitizer` Consumer subscribes, redacts PII, and writes the sanitized document back via `attachSanitized`. The same Consumer then starts an `ExtractionWorkflow` instance. The workflow's `extractStep` calls `ExtractionAgent` — the single AutonomousAgent — with the target schema definition as `TaskDef.instructions(...)` and the sanitized document as a `TaskDef.attachment(...)`. The agent's `after-llm-response` guardrail (`RecordGuardrail`) validates each candidate response. Once a record passes, the workflow writes `RecordExtracted`. `ExtractionView` projects every entity event into a read-model row; `ExtractionEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The guardrail (`RecordGuardrail`) is a deterministic validator — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct moments where the system waits:

1. The `DocumentSanitizer` subscription lag between `DocumentSubmitted` and `DocumentSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `ExtractionJobEntity` every 1 s up to its 15 s timeout, advancing as soon as `job.sanitized().isPresent()` returns true.

The agent call itself is bounded by `extractStep`'s 60 s timeout. If the guardrail rejects a candidate record, the agent loop retries within the same task, consuming one of its 3 iterations.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → EXTRACTING → RECORD_EXTRACTED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error (or guardrail-exhaustion after 3 iterations) during `EXTRACTING`. A `FAILED` job's prior data is preserved on the entity.
- There is no post-extraction approval state. Extraction is fully automated; the deployer decides what to do with the `ExtractedRecord` downstream.

## Entity model

`ExtractionJobEntity` is the source of truth. It emits five event types. `ExtractionView` projects every event into a row used by the UI. `DocumentSanitizer` subscribes to entity events to compute the sanitized form. `ExtractionWorkflow` both reads (`getJob`) and writes (`markExtracting`, `recordExtracted`, `fail`) on the entity. The relationship between `ExtractionAgent` and `ExtractedRecord` is "returns" — the agent's task result is the record.

## Governance flow

For any record that lands in the entity log, the document passed through:

1. **PII sanitizer** — the model never sees raw identifiers; the audit log retains the raw form.
2. **ExtractionAgent** — one model call, one schema-bound output.
3. **after-llm-response guardrail** — bad parses, unknown field names, absent required fields, and over-length values are caught before the response leaves the agent loop.

Each step is independent. Removing one opens an explicit gap the other does not silently cover.
