# Architecture — rag-support-bot

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ConversationEndpoint` accepts a customer query and writes a `QueryReceived` event onto `ConversationEntity`. The `MessageSanitizer` Consumer subscribes, strips PII, and writes the sanitized query back via `attachSanitized`. The same Consumer starts a `ConversationWorkflow` instance. The workflow's `retrieveStep` queries `KnowledgeBase` entity instances in-process to collect the top-5 passages relevant to the sanitized query. The `replyStep` calls `SupportAgent` — the single AutonomousAgent — with the sanitized query as `TaskDef.instructions(...)` and the retrieved passages as `TaskDef.attachment("passages.txt", ...)`. The agent's `before-agent-response` guardrail (`ReplyGuardrail`) validates each candidate response. Once a reply passes, the workflow writes `ReplyRecorded` and runs `GroundingScorer` in `evalStep`. The grounding eval lands as `EvaluationScored`. `ConversationView` projects every entity event into a read-model row; `ConversationEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`GroundingScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

The `KnowledgeBase` is also an EventSourcedEntity, one instance per article. This keeps retrieval self-contained (no external vector database) while still following the Akka component model. Passage segmentation happens deterministically in the event-applier when an article is loaded.

## Interaction sequence

The sequence traces the happy path (J1). Note three distinct pauses:

1. The `MessageSanitizer` subscription lag between `QueryReceived` and `QuerySanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop — polls `ConversationEntity` every 1 s up to its 15 s timeout, advancing as soon as `conversation.sanitized().isPresent()` returns true.
3. The agent call itself, bounded by `replyStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds.

## State machine

Seven states for `ConversationEntity`. The interesting paths:

- The happy path is `RECEIVED → SANITIZED → RETRIEVING → REPLYING → REPLY_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `RECEIVED`, and an agent error (or guardrail-exhaustion) during `REPLYING`. A `FAILED` conversation's prior data is preserved on the entity — the UI shows the partial state for support managers reviewing the log.
- There is no `RESOLVED` or `CLOSED` state. The reply is delivered; whether the customer's problem is resolved is tracked outside this system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`ConversationEntity` is the source of truth. It emits seven event types. `ConversationView` projects every event into a row used by the UI. `MessageSanitizer` subscribes to entity events to compute the sanitized form. `ConversationWorkflow` both reads (`getConversation`) and writes (`markReplying`, `recordRetrieval`, `recordReply`, `recordEvaluation`, `fail`) on the entity. `KnowledgeBase` is a separate entity family that holds the indexed article content; the workflow reads from it during `retrieveStep` but the knowledge base does not subscribe to conversation events. The relationship between `SupportAgent` and `SupportReply` is "returns" — the agent's task result is the reply record.

## Defence-in-depth governance flow

For any reply that lands in the entity log, the customer query passed through:

1. **PII sanitizer** — the model never sees customer identifiers; the audit log retains the raw form.
2. **SupportAgent** — one model call, one structured output, grounded in retrieved passages.
3. **before-agent-response guardrail** — uncited claims, escalation-flag mismatches, and out-of-bounds reply lengths are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed reply still gets a 1–5 grounding score so support managers know which replies to spot-check.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover. In particular, the guardrail does not replace the sanitizer (the guardrail only sees the redacted form), and the evaluator does not replace the guardrail (the evaluator runs after the reply is already recorded).
