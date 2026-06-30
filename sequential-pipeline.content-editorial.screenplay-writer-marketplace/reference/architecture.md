# Architecture — screenplay-writer-marketplace

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `ScreenplayEndpoint` receives a `{sourceTitle, sourceText}` POST. Before any entity write, `PiiSanitizer` runs synchronously on the submitted text — email addresses, phone numbers, and display names from email headers are replaced with stable numbered placeholders. The sanitized text and the placeholder map are passed to `ScreenplayEntity.create()`, which emits `ScreenplayCreated`. `ScreenplayEndpoint` then starts `ScreenplayPipelineWorkflow` keyed by `"pipeline-" + screenplayId`.

The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `ScreenplayAgent` with `TaskDef.taskType(PARSE_SOURCE)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.extractCharacters`, `extractSettings`, and `extractBeats` on the sanitized text. Once the agent returns a `ParsedSource`, the workflow writes `SourceParsed` onto the entity and advances to `developStep` — same pattern, the DEVELOP task carries `phase = DEVELOP`. Then `formatStep` runs with `phase = FORMAT`.

After the FORMAT task completes, `DeliveryGuardrail` intercepts the agent's response before it is returned to the workflow. The guardrail reads the placeholder map from the entity and scans the formatted screenplay text for any raw PII tokens. On pass, the workflow writes `ScreenplayFormatted` then `ScreenplayDelivered`. On block, it writes `DeliveryBlocked` and the entity transitions to `DELIVERY_BLOCKED`.

`ScreenplayView` projects every event into a read-model row. `ScreenplayEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `PiiSanitizer` and `DeliveryGuardrail` are deterministic in-process components — that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The sanitizer runs at the boundary between the user and the system — before any entity write, before any agent call. The agent never receives raw PII; it works entirely with placeholders. This is by design: asking the agent to "ignore PII" or "redact names" introduces non-determinism. The sanitizer is deterministic.
2. The delivery guardrail runs at the other boundary — between the agent's output and the workflow's final write. This is the only place in the pipeline where the original PII values are compared against the output. The guardrail does not modify the output; it either passes it or blocks it entirely.

The agent calls themselves are bounded by per-step timeouts (60 s on parse / develop / format). `deliverStep` is synchronous and finishes in milliseconds.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → PARSING → PARSED → DEVELOPING → DEVELOPED → FORMATTING → FORMATTED → DELIVERED`.
- Three failure transitions land in `FAILED`: an agent error during PARSING, DEVELOPING, or FORMATTING (exhausted retries or step timeout).
- One guardrail transition lands in `DELIVERY_BLOCKED`: the FORMAT task completed but the guardrail detected a residual PII token in the output. `DELIVERY_BLOCKED` is a terminal state. The screenplay is preserved on the entity for review; the blocked tokens are recorded in `DeliveryBlocked.detectedTokens`.
- There is no `APPROVED` or `PUBLISHED` state. The screenplay is a creative artifact; the author reviews and acts outside the system.

## Entity model

`ScreenplayEntity` is the source of truth. It emits ten event types across the lifecycle. `ScreenplayView` projects every event into a row used by the UI. `ScreenplayPipelineWorkflow` both reads (via `getScreenplay`) and writes (via the command handlers) on the entity. `ScreenplayAgent` returns three typed results — `ParsedSource`, `ScenePlan`, `Screenplay` — each of which becomes the payload of a matching entity event.

## Defence-in-depth governance flow

For any screenplay that lands in the entity log, the source material passed through:

1. **PII sanitizer (ingestion)** — before any entity write or agent call. Every PII token in the source is replaced by a stable placeholder. The agent never sees raw PII.
2. **ScreenplayAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Delivery guardrail (egress)** — before `ScreenplayDelivered` is written. The formatted screenplay is checked against the original PII values from the placeholder map. Any match blocks delivery.

Each mechanism is independent. The sanitizer does not check whether the output is clean — it only cleans the input. The guardrail does not sanitize — it only passes or blocks. Removing either opens an explicit gap the other does not silently cover.
