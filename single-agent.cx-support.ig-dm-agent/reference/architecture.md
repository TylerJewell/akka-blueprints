# Architecture — ig-dm-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `DmEndpoint` accepts an inbound message submission and writes a `MessageReceived` event onto `DmEntity`. The `MessageSanitizer` Consumer subscribes, strips PII from the raw DM text, and writes the sanitized form back via `attachSanitized`. The same Consumer then starts a `DmWorkflow` instance. The workflow's `replyStep` calls `DmReplyAgent` — the single AutonomousAgent — with the brand voice profile as `TaskDef.instructions(...)` and the sanitized message as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ReplyGuardrail`) validates each candidate reply against the brand profile rules. Once a reply passes, the workflow writes `ReplyRecorded`. `DmView` projects every entity event into a read-model row; `DmEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. Brand-compliance checking (`ReplyGuardrail`) is deterministic rule-based code — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments where the system hands off asynchronously:

1. The `MessageSanitizer` subscription lag between `MessageReceived` and `MessageSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `DmEntity` every 1 s up to its 15 s timeout, advancing as soon as `message.sanitized().isPresent()` returns true.

The agent call is bounded by `replyStep`'s 60 s timeout. A guardrail rejection increments the iteration counter and triggers an in-loop retry without leaving the agent's task context — no external round-trip occurs.

## State machine

Five states. The paths:

- The happy path is `RECEIVED → SANITIZED → REPLYING → REPLIED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent error (or guardrail-iteration exhaustion) during `REPLYING`. A `FAILED` message's prior data is preserved on the entity — the UI shows the partial state for a moderator.
- There is no `DISPATCHED` or `DELIVERED` state. The blueprint produces the approved reply text; actual dispatch to Instagram's API is a deployer integration step outside this sample.

## Entity model

`DmEntity` is the source of truth. It emits five event types. `DmView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized form. `DmWorkflow` both reads (`getMessage`) and writes (`markReplying`, `recordReply`, `fail`) on the entity. The relationship between `DmReplyAgent` and `DmReply` is "returns" — the agent's task result is the reply record.

## Defence-in-depth governance flow

For any reply that lands in the entity log, the inbound message passed through:

1. **PII sanitizer** — the model never sees customer identifiers; the audit log retains the raw form.
2. **DmReplyAgent** — one model call, one structured output.
3. **before-agent-response guardrail** — prohibited terms, competitor mentions, over-length drafts, and out-of-scope commitments are caught before the response leaves the agent loop.

Each step is independent. Removing one opens an explicit gap the other does not silently cover.
