# Architecture — shared-memory-multi-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`MemoryEndpoint` is the entry point. A submitted block seed is logged on `MemoryEventLog` and written to `MemoryBlockEntity` in one command pair. `Bootstrap` starts one `WriteWorkflow` per registered writer id. Each write loop reads the current block snapshot from `MemoryBlockView`, calls its `WriterAgent` instance, passes the result through the G1 guardrail and the S1 sanitizer, and commits the fragment to `FragmentEntity` while incrementing the block's pending count.

`FragmentConsumer` watches `FragmentEntity` events; when a block's pending count crosses the threshold, it starts a `ConsolidationWorkflow` for that block. The workflow calls `ConsolidatorAgent`, commits the merged `ConsolidatedSnapshot` to `MemoryBlockEntity`, and marks each merged fragment on `FragmentEntity`. Both entity types project into `MemoryBlockView` — the shared read model every writer and the UI queries.

`ConsolidationScheduler` and `FragmentAgeMonitor` sit alongside as two background TimedActions: the scheduler triggers consolidation on blocks that have accumulated fragments without hitting the threshold; the age monitor re-queues fragments that have been `PENDING` too long so consolidation is never permanently stalled. `AgentRegistry` (a key-value entity) holds the writer roster used by `Bootstrap` and exposed through the API.

## Interaction sequence

The sequence diagram traces the happy path (J1): seed a block → writers each contribute a fragment (every write passing the guardrail and sanitizer) → fragment count reaches threshold → `FragmentConsumer` starts `ConsolidationWorkflow` → `ConsolidatorAgent` merges fragments → block moves to `CONSOLIDATED` → the view projects the new snapshot → the UI receives an SSE update. Writer loops are already polling before any fragment exists; the only thing seeding the block adds is the initial snapshot for them to build on.

## State machine

`MemoryBlockEntity` is the coordination anchor. `SEEDED` is the initial state — the block has content but no agent has written to it yet. The first fragment write activates the block to `ACTIVE`. When consolidation starts, the block enters `CONSOLIDATING`; during this window the G1 guardrail refuses new writes to prevent a race between writers and the consolidator. On a successful consolidation commit the block returns to `CONSOLIDATED` — from which a new writer fragment re-activates it to `ACTIVE` to start the next cycle. An operator `archive` command terminates either `ACTIVE` or `CONSOLIDATED` blocks.

## Entity model

`MemoryBlockEntity` is the source of truth for the shared context; every state transition emits one of five event types. `FragmentEntity` is the source of truth for each individual contribution; five event types track its lifecycle from submission through sanitization to merge or rejection. `MemoryBlockView` combines projections from both entities into one queryable read model — the single surface agents and the UI query. `MemoryEventLog` is the audit trail of block creation and consolidation trigger events.

## Concurrency and timeouts

- Write-side concurrency is managed by `MemoryBlockEntity`'s single-writer guarantee. While the block is `CONSOLIDATING`, the G1 guardrail refuses write calls, preventing a fragment from arriving mid-consolidation.
- Per-step timeouts: `WriteWorkflow.contributeStep` uses 90 s; `ConsolidationWorkflow.consolidateStep` uses 120 s. Both call agents whose LLM latency can exceed the default 5 s (Lesson 4).
- Writer loops are staggered by a 30 s inter-cycle timer so all three writers do not simultaneously query the view and post fragments.
- `FragmentConsumer` de-duplicates consolidation starts by using the `blockId` as the `ConsolidationWorkflow` id; a duplicate start is a no-op.
- `FragmentAgeMonitor` prevents permanent pending accumulation by re-queuing fragments older than three minutes, ensuring the next `ConsolidationScheduler` tick picks them up.
- The view exposes no WHERE filter on enum columns; callers filter status client-side (Lesson 2).
