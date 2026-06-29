# Architecture — ghostwriter

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `DraftEndpoint` accepts a submission and writes a `BriefSubmitted` event onto `DraftEntity`. The `CorpusSanitizer` Consumer subscribes, redacts PII from every writing sample, and writes the sanitized corpus back via `attachSanitizedCorpus`. The same Consumer then starts a `DraftWorkflow` instance. The workflow's `draftStep` calls `GhostwriterAgent` — the single AutonomousAgent — with the writing brief as `TaskDef.instructions(...)` and each sanitized sample as a separate `TaskDef.attachment(...)`. The agent's `after-llm-response` guardrail (`DraftOutputGuardrail`) validates each candidate response before it commits. Once a draft passes, the workflow writes `DraftReady` to the entity. `DraftView` projects every entity event into a read-model row; `DraftEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `DraftOutputGuardrail` is a validation hook registered on the agent's response channel — not an autonomous component. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct pauses occur:

1. The `CorpusSanitizer` subscription lag between `BriefSubmitted` and `CorpusSanitized` — typically sub-second in process.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `DraftEntity` every 1 s up to its 15 s timeout, advancing as soon as `draft.sanitizedCorpus().isPresent()` returns true.

The agent call itself is bounded by `draftStep`'s 90 s timeout, which accommodates reading multiple sample attachments plus LLM inference latency.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → SANITIZED → DRAFTING → DRAFT_READY`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SUBMITTED`, and an agent error or guardrail-iteration-exhaustion during `DRAFTING`. A failed draft's prior data is preserved on the entity for the user to inspect.
- There is no `PUBLISHED` state. The draft is advisory; the human edits and publishes outside the system. This blueprint deliberately stops at `DRAFT_READY`.

## Entity model

`DraftEntity` is the source of truth. It emits five event types. `DraftView` projects every event into a row used by the UI. `CorpusSanitizer` subscribes to entity events to compute the sanitized corpus. `DraftWorkflow` both reads (`getDraft`) and writes (`markDrafting`, `recordDraft`, `fail`) on the entity. The relationship between `GhostwriterAgent` and `DraftResult` is "returns" — the agent's task result is the draft record.

## Governance flow

For any draft that lands in the entity log, the writing samples passed through:

1. **PII sanitizer** — the model never sees personal identifiers; the audit log retains the raw form.
2. **GhostwriterAgent** — one model call, one structured output.
3. **after-llm-response guardrail** — verbatim-reproduction runs, leaked redaction markers, out-of-range fidelity scores, and empty bodies are caught before the response commits to the workflow.

Each step is independent. Removing one opens an explicit gap the other does not silently cover.
