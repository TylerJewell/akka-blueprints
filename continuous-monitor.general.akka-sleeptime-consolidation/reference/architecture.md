# Architecture — sleeptime-consolidation

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`ConsolidationTrigger` is the heartbeat — a TimedAction that ticks every 30 s, queries `MemoryView` for sessions whose `stepsSinceLastConsolidation` has reached the configured threshold (default: 5), and starts a `ConsolidationWorkflow` for each eligible `MemoryBlockEntity`. A second TimedAction, `DriftEvalRunner`, ticks independently every 30 minutes and scores recently consolidated blocks for semantic drift.

`InteractionCounterEntity` tracks step counts per session. `MemoryEndpoint`'s `POST /sessions/{sessionId}/step` call increments the counter; `ConsolidationTrigger` reads it via `MemoryView` on each tick.

`PrimaryAgent` reads the current consolidated block directly from `MemoryBlockEntity` when answering a user query. It never writes.

## Interaction sequence

The sequence traces the J1 happy path: counter reaches threshold → trigger starts workflow → load → consolidate → commit → operator sees the event in the SSE stream. Two distinct async gaps: (1) the natural ~30 s tick gap between threshold being reached and `ConsolidationTrigger` firing, and (2) the `consolidateStep` LLM call (up to 60 s; times out to STALE if exceeded).

The `Note over B,U: SSE streams ConsolidationStarted to operator` block marks the HOTL observation point — this is where an operator watching the App UI's activity stream first sees the consolidation beginning.

## State machine

Four states with a cycle for active blocks. The interesting paths:

- ACTIVE → CONSOLIDATING: triggered by `ConsolidationStarted` from `loadStep`. The block remains readable by `PrimaryAgent` during consolidation (optimistic reads; the prior version is returned until `BlockConsolidated` is applied).
- CONSOLIDATING → STALE: triggered by a `consolidateStep` timeout. The block is retired; the operator is notified via the HOTL stream.
- CONSOLIDATED → ACTIVE: triggered by `BlockUpdated` when new content is appended via `POST /blocks/{blockId}/content`. This resets `stepsSinceLastConsolidation` so the next consolidation cycle can begin.

## Entity model

`MemoryBlockEntity` is the source of truth for all block state. `InteractionCounterEntity` tracks per-session step counts and is projected into `MemoryView` alongside block rows so `ConsolidationTrigger` has a single query surface.

## Three-layer governance

For any memory rewrite that the primary agent subsequently uses:

1. **before-tool-call guardrail** (G1) — only a `ConsolidationWorkflow` with an active lease may call `writeMemoryBlock`. All other callers receive a structured rejection and a `GuardrailTriggered` event is emitted.
2. **HOTL deployer-runtime-monitoring** (H1) — every consolidation event, guardrail trip, and drift score is streamed to the operator via SSE. The operator can observe the system in real time without blocking any workflow.
3. **eval-periodic drift-fairness-watch** (E1) — `DriftEvalRunner` compares each consolidated block against its prior version, scoring semantic drift 0–100. Blocks with drift ≥ 70 are flagged in the UI for human review before the primary agent depends on them.

Each mechanism is independent. Removing any one leaves the others intact but opens a specific risk: removing G1 allows uncontrolled writes; removing H1 makes consolidation opaque; removing E1 allows silent meaning drift to accumulate undetected.
