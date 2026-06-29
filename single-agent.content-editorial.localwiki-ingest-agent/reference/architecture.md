# Architecture — localwiki-ingest-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `IngestEndpoint` accepts a submission and writes a `SourceSubmitted` event onto `IngestEntity`. The `ContentSanitizer` Consumer subscribes, fetches the source, redacts PII, and writes the sanitized content back via `attachSanitized`. The same Consumer then starts an `IngestWorkflow` instance. The workflow's `ingestStep` calls `WikiIngestAgent` — the single AutonomousAgent — with the target namespace in the filing instructions and the sanitized content as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`WritePageGuardrail`) intercepts every proposed `write_wiki_page` tool call. Once a call passes, the agent returns a `WikiPage`; the workflow writes `PageFiled` to the entity. `WikiPageView` projects every entity event into a read-model row; `IngestEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `WritePageGuardrail` is a supporting class registered on the agent — it inspects proposed tool calls but makes no LLM call of its own. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct points where the system waits on something:

1. The `ContentSanitizer` subscription lag between `SourceSubmitted` and `ContentSanitized` — sub-second in normal operation; the fetch simulation adds a bounded delay.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `IngestEntity` every 1 s up to its 15 s timeout, advancing as soon as `ingest.sanitized().isPresent()` returns true.

The agent call itself is bounded by `ingestStep`'s 60 s timeout. The guardrail runs synchronously inside the agent loop before each tool execution; it completes in microseconds.

## State machine

Five states. The notable paths:

- The happy path is `SUBMITTED → SANITIZED → INGESTING → PAGE_FILED`.
- Two failure transitions land in `FAILED`: a fetch or sanitizer error during `SUBMITTED`, and an agent error (including guardrail-exhaustion after 3 iterations) during `INGESTING`. A `FAILED` ingest's prior data is preserved on the entity — the UI shows the partial state and the failure reason.
- There is no `APPROVED` state. The agent's decision to file a page is autonomous; oversight is human-on-loop. A deployer requiring human approval would add a command and an `AWAITING_APPROVAL` state between `PAGE_FILED` and a terminal `PUBLISHED` state.

## Entity model

`IngestEntity` is the source of truth. It emits five event types. `WikiPageView` projects every event into a row used by the UI. `ContentSanitizer` subscribes to entity events to compute the sanitized form and start the workflow. `IngestWorkflow` both reads (`getIngest`) and writes (`markIngesting`, `recordPage`, `fail`) on the entity. The relationship between `WikiIngestAgent` and `WikiPage` is "returns" — the agent's task result is the page record.

## Defence-in-depth governance flow

For any page that lands in the entity log, the content passed through:

1. **PII sanitizer** — the model never sees identifiers; the audit log retains the raw fetched text.
2. **WikiIngestAgent** — one model call, one structured tool call producing the page.
3. **before-tool-call guardrail** — path traversal attempts, namespace violations, and empty titles are caught before the tool executes, before any byte is written to the wiki store.

Each step is independent. Removing one opens an explicit gap the other does not silently cover.
