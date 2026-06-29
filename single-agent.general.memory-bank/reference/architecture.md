# Architecture — memory-bank

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `MemoryEndpoint` accepts a submission and writes a `MemoryEntrySubmitted` event onto `MemoryEntity`. The `MemorySanitizer` Consumer subscribes, redacts PII, and writes the sanitized content back via `attachSanitized`. The same Consumer then starts a `MemoryWorkflow` instance. The workflow's `storeStep` calls `MemoryAgent` — the single AutonomousAgent — with either a `REMEMBER_CONTENT` or `RECALL_CONTENT` task. For recall operations, the workflow fetches stored entries from `MemoryView` and injects them into the task instructions before calling the agent; this keeps context retrieval inside the workflow layer, not in a second agent. Once the agent returns a result, the workflow writes `MemoryEntryStored`. `MemoryView` projects every entity event into a read-model row; `MemoryEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The recall context injection is a workflow-level string construction — not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path for a REMEMBER operation (J1). The RECALL path follows the same structure through sanitize and workflow, differing only in the task type the workflow dispatches and the result type the agent returns.

Two distinct moments where the system waits:

1. The `MemorySanitizer` subscription lag between `MemoryEntrySubmitted` and `MemoryEntrySanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `MemoryEntity` every 1 s up to its 15 s timeout, advancing as soon as `memory.sanitized().isPresent()` returns true.

The agent call itself is bounded by `storeStep`'s 30 s timeout, which accommodates both the lightweight REMEMBER path and the heavier RECALL path (larger instruction payload when stored entries are injected).

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → STORING → STORED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error during `STORING`. A `FAILED` entry's prior data is preserved on the entity.
- `STORED` is the terminal happy state. Memory entries do not have an EXPIRED or DELETED state in this baseline — expiry and deletion are deployer concerns addressed by extending the entity commands.

## Entity model

`MemoryEntity` is the source of truth. It emits five event types. `MemoryView` projects every event into a row used by the UI and by the workflow's RECALL context injection. `MemorySanitizer` subscribes to entity events to compute the sanitized form. `MemoryWorkflow` both reads (`getMemory`) and writes (`markStoring`, `recordResult`, `fail`) on the entity. `MemoryAgent` returns either a `StoreConfirmation` (REMEMBER) or a `RecallResult` (RECALL) — both are modelled as result types of the `MemoryEntryStored` event payload.

## Sanitizer-first governance

For any entry that lands in the entity log, the content passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw form.
2. **MemoryAgent** — one model call per operation, typed output.

The sanitizer is the only governance mechanism in this baseline. Deployers who need structural-output validation (guardrail) or an on-decision evaluator for recall quality can extend `eval-matrix.yaml` with additional controls following the pattern established by the docreview blueprint's G1 and E1 controls.
