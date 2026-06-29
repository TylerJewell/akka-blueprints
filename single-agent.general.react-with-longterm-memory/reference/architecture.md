# Architecture — mem0-react-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call and two independent governance paths. `AgentEndpoint` accepts a turn and writes a `TurnStarted` event onto `SessionEntity`, then invokes `Mem0ReactAgent` — the single AutonomousAgent. The agent's ReAct loop calls four tools: `recall-memories` reads from `MemoryView`, `store-memory` calls `MemoryEntity.requestStore`, `web-search` hits a stub, and `calculate` evaluates an expression inline. The store-memory path triggers `FactStoreRequested` on `MemoryEntity`, which the `FactSanitizer` Consumer picks up, redacts PII, and writes `FactSanitized` back. `MemoryWriteWorkflow` then executes `persistStep`, writing `FactPersisted`. In parallel, `DriftMonitor` listens to `FactPersisted` events and emits `MemoryDriftSignaled` when a user's fact count crosses the configured threshold. Both `SessionView` and `MemoryView` project entity events for the UI. `AgentEndpoint` serves both views over REST and SSE.

The graph has no second agent. `DriftMonitor` is a rule-based Consumer — a counter, not an LLM. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces two sessions (J1 happy path). Two moments where the system stores state asynchronously:

1. The `FactSanitizer` subscription lag between `FactStoreRequested` and `FactSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside `MemoryWriteWorkflow` — polls `MemoryEntity` every 1 s up to its 15 s timeout, advancing as soon as `memory.sanitized().isPresent()` returns true.

The agent turn is bounded by `ANSWER_TURN`'s `maxIterationsPerTask(8)`, which accommodates a full ReAct chain (recall → reason → tool → reason → answer) within a single task execution.

## State machine

Four states for `MemoryEntity`. The key paths:

- The happy path is `SANITIZE_PENDING → SANITIZED → PERSISTED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `SANITIZE_PENDING`, and a persist error during `SANITIZED`. A `FAILED` fact retains its raw request on the entity — the audit log has the original text even if the clean path failed.
- `PERSISTED` is a terminal state. Facts are immutable once persisted; updates create a new fact entity.

`SessionEntity` has fewer states: `OPEN → ACTIVE → CLOSED`. It is always in `ACTIVE` while a turn is in flight.

## Entity model

`SessionEntity` holds the conversation thread; `MemoryEntity` holds one fact each. `SessionView` and `MemoryView` project their respective entity streams. `FactSanitizer` and `DriftMonitor` subscribe to `MemoryEntity` events for their respective governance roles. `MemoryWriteWorkflow` both reads (`getMemory`) and writes (`attachSanitized`, `persist`, `fail`) on `MemoryEntity`. The relationship between `Mem0ReactAgent` and `AgentAnswer` is "returns" — the agent's task result is the answer record.

## Governance flow

For any fact that lands in long-term memory, the text passed through:

1. **Agent's store-memory tool call** — the user's raw words, verbatim, as the agent heard them.
2. **PII sanitizer** — identifiers are removed before the fact leaves the `MemoryEntity`'s sanitize-pending state; the raw text is preserved on the entity for audit but never appears in recall queries.
3. **MemoryWriteWorkflow.persistStep** — only `sanitized.cleanText` is written to the persistent read model; callers who query `MemoryView` never see the raw form.

The drift monitor runs orthogonally: it does not inspect fact content but counts events per user, signalling accumulation for deployer review without touching the agent loop.

Each governance component is independent. The sanitizer does not know about the drift monitor; the drift monitor does not know about the sanitizer. Removing either one opens a gap the other does not silently cover.
