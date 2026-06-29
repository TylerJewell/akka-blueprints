# Architecture — wp-autotagger

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `PostEndpoint` accepts a submission and writes a `PostIngested` event onto `PostEntity`. The `WpApiConsumer` Consumer subscribes, calls the in-process `WpApiStub` to fetch the post body, and writes it back via `attachBody`. The same Consumer then starts a `TaggingWorkflow` instance. The workflow's `tagStep` calls `TagProposerAgent` — the single AutonomousAgent — with the tagging configuration as `TaskDef.instructions(...)` and the post body as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`TagProposalGuardrail`) validates each candidate `applyTags` tool call before it executes. Once a valid tool call goes through, `WpApiStub` records the applied tags and the workflow writes `TagsApplied` to the entity. `PostView` projects every entity event into a read-model row; `PostEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The `WpApiStub` is a deterministic in-process component — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system pauses:

1. The `WpApiConsumer` subscription lag between `PostIngested` and `PostBodyFetched` — sub-second in normal operation, as the stub reads from a local JSONL file.
2. The `awaitBodyStep` polling loop inside the workflow — polls `PostEntity` every 1 s up to its 15 s timeout, advancing as soon as `post.body().isPresent()` returns true.

The agent call itself is bounded by `tagStep`'s 60 s timeout. The `WpApiStub.applyTags` call is synchronous and finishes in milliseconds.

## State machine

Five states. The interesting paths:

- The happy path is `INGESTED → BODY_FETCHED → TAGGING → TAGS_APPLIED`.
- Two failure transitions land in `FAILED`: a fetch error during `INGESTED` (the stub cannot resolve the URL), and an agent error (or guardrail-exhaustion) during `TAGGING`. A `FAILED` job's prior data is preserved on the entity — the UI shows the partial state.
- There is no `APPROVED` or `PENDING_REVIEW` state. Tags are applied automatically; human review happens outside the system by inspecting the published post.

## Entity model

`PostEntity` is the source of truth. It emits five event types. `PostView` projects every event into a row used by the UI. `WpApiConsumer` subscribes to entity events to fetch and attach the post body. `TaggingWorkflow` both reads (`getPost`) and writes (`markTagging`, `applyTags`, `fail`) on the entity. The relationship between `TagProposerAgent` and `TagProposal` is "returns" — the agent's task result is the proposal record.

## Governance flow

For any tag proposal that lands in the entity log, the post went through:

1. **WpApiConsumer** — the post body is fetched and any credential-like tokens are redacted before the agent sees the text.
2. **TagProposerAgent** — one model call, one structured tool invocation.
3. **before-tool-call guardrail** — wrong post IDs, over-count tag lists, and malformed tag strings are caught before the `applyTags` tool call executes.

Each step is independent. The guardrail at the tool boundary is the primary governance layer; the body-redaction step ensures the model never receives live credential material even if a post author accidentally embedded one.
