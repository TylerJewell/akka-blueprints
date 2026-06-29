# Architecture — long-term-memory-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one conversational agent backed by two entity types. `ConversationEndpoint` accepts a new session turn and writes it to `SessionEntity`. At the same time the endpoint starts a `MemoryExtractionWorkflow` for that turn. The workflow's `recallStep` queries `MemoryView` — a projection of `MemoryEntity` events — to retrieve the user's top-K memories and assembles them into a `memories.json` attachment. The `respondStep` calls `MemoryAgent` — the single AutonomousAgent — passing the user message as task instructions and the recalled memories as a task attachment. The agent returns an `AgentReply` containing both the response text and candidate memory entries. The workflow writes `ReplyRecorded` and `MemoryExtracted` events onto `SessionEntity`. The `MemorySanitizer` Consumer subscribes to `MemoryExtracted` events, scrubs PII from each raw memory entry, and writes sanitized `MemoryEntry` records to `MemoryEntity`. Both entities project into views; `ConversationEndpoint` serves those views to the UI over REST and SSE.

There is no second agent. The `MemorySanitizer` is a deterministic regex-and-heuristic pipeline — it talks to no model. That is what makes this a faithful **single-agent** blueprint: exactly one component calls a model, and the governance mechanism wraps what comes out of that call, not what goes in.

## Interaction sequence

The sequence traces the happy path (J1) for a first turn. Two things to note:

1. The `recallStep` returns an empty memory list on the very first turn. That is a valid state: the agent generates a reply with no prior context and extracts any new facts worth storing.
2. The `MemorySanitizer` runs asynchronously after the workflow completes — the user's reply arrives before memory storage is confirmed. The UI receives a separate `MemoryStored` SSE event when persistence finishes.

For subsequent turns the recall step returns populated memories and the agent's reply incorporates them. The sequence is otherwise identical.

## State machine

Three states for `SessionEntity`. The machine is intentionally bounded:

- `ACTIVE` handles all normal turn processing. A failed turn (`TurnFailed`) leaves the session in `ACTIVE` — the session itself stays open; only that turn is marked as failed.
- `CLOSING` is a transient state the system enters when the 50-turn cap is reached. It gives any in-flight workflow a clean landing zone before the entity transitions to `CLOSED`.
- `CLOSED` is terminal. A closed session's history is readable; no further turns are accepted. The user's `MemoryEntity` is unaffected — memories from a closed session persist and are recalled in future sessions.

`MemoryEntity` has no explicit state machine depicted here. Its lifecycle is append-only: entries are stored or expired; the entity never transitions to a terminal state.

## Entity model

`SessionEntity` is the event hub for conversation state. It emits six event types, three of which drive other components: `MemoryExtracted` drives `MemorySanitizer`, and all events project into `SessionView`. `MemoryEntity` is the durable memory store; its `MemoryStored` events project into `MemoryView`, which `MemoryExtractionWorkflow` queries at the start of each turn. `MemoryAgent` sits outside both entities — it interacts only through the workflow, returning an `AgentReply` that the workflow distributes to the appropriate entity commands.

## Governance flow

For any memory entry that lands in `MemoryEntity`, the content has passed through:

1. **MemoryAgent** — the one model call produces both the reply and the raw candidate memories.
2. **MemorySanitizer** — PII is redacted before any raw text reaches `MemoryEntity`. The sanitized form is the only version stored.
3. **MemoryView** — only the sanitized form is projected into the view and therefore the only form recalled on future turns.

The raw `RawMemoryEntry` list exists only in-flight, inside the `MemoryExtracted` event payload and the `MemorySanitizer`'s processing loop. It is never written to durable state.
