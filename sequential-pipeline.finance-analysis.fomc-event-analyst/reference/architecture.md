# Architecture — fomc-event-analyst

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `FomcEndpoint` accepts an `{eventId}` POST, writes `AnalysisCreated` onto `FomcEventEntity`, and starts `FomcPipelineWorkflow` keyed by `"pipeline-" + analysisId`. The workflow's first step (`gatherStep`) emits `GatherStarted`, then calls `FomcAnalystAgent` with `TaskDef.taskType(GATHER_MARKET_DATA)` and a `phase = GATHER` metadata tag. The agent invokes `GatherTools.fetchIndicators` and `GatherTools.fetchYieldCurve`; both calls are in-process and return deterministic offline data from the classpath sample files. Once the agent returns a `MarketSnapshot`, the workflow writes `MarketDataGathered` onto the entity and advances to `interpretStep` — the INTERPRET task carries `phase = INTERPRET` and receives the serialized `MarketSnapshot` as its instruction context. Then `synthesizeStep` runs with `phase = SYNTHESIZE`. Before the agent's `PolicyAnalysis` candidate is accepted, `FinancialOutputGuardrail` reviews it: checks signal attribution, market-indicator grounding, and non-emptiness, then either accepts or returns a structured rejection to the agent loop. On accept, the workflow writes `AnalysisSynthesized` and `OutputReviewed` onto the entity. `PolicyAnalysisView` projects every event into a read-model row; `FomcEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component: `FomcAnalystAgent`. The guardrail is a deterministic quality check, not an LLM call, which preserves the single-agent sequential-pipeline invariant.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `gatherStep` and `interpretStep`, the workflow writes `MarketDataGathered` onto the entity. The next step then reads `snapshot` from the entity to build the INTERPRET task's instruction context. The agent never sees gather-phase context inside the interpret task's conversation; the typed handoff is the only path information travels.
2. The `before-agent-response` guardrail fires on the SYNTHESIZE task's candidate response — not on individual tool calls. This is the right hook for financial-output quality: the agent may call `draftPolicySection` correctly on every iteration, yet still produce a final `PolicyAnalysis` that references an indicator not present in the recorded snapshot. The guardrail catches this at the response boundary, before the event is committed.

The agent calls are bounded by per-step timeouts (60 s on gather / interpret / synthesize). `reviewStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → GATHERING → GATHERED → INTERPRETING → INTERPRETED → SYNTHESIZING → SYNTHESIZED → REVIEWED`.
- Three failure transitions land in `FAILED`: an agent error during `GATHERING`, `INTERPRETING`, or `SYNTHESIZING`. A `FAILED` analysis's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `APPROVED` state. The analysis is advisory output for a human analyst; the blueprint stops at `REVIEWED`.

## Entity model

`FomcEventEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the output review, the guardrail audit, the failure, and the initial creation. `PolicyAnalysisView` projects every event into a row used by the UI. `FomcPipelineWorkflow` both reads (`getAnalysis`) and writes (`startGather`, `recordSnapshot`, `startInterpret`, `recordSignalSet`, `startSynthesize`, `recordAnalysis`, `recordReview`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `FomcAnalystAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any analysis that lands in the entity log, the FOMC event identifier passed through:

1. **FomcAnalystAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase. The GATHER and INTERPRET tasks return structured data; the SYNTHESIZE task produces the final advisory output.
2. **Financial output guardrail** — the SYNTHESIZE task's candidate `PolicyAnalysis` is reviewed before it is committed. Signal attribution, market-indicator grounding, and non-emptiness are each checked; any failure is returned to the agent loop as a structured rejection. The guardrail does not check phase ordering (there are no cross-phase tool calls in this blueprint); it checks output quality.

Each layer covers a distinct failure mode. The task-boundary handoffs ensure the agent is never reasoning over stale or absent data. The guardrail ensures the final output cannot contain ungrounded financial claims. Removing either layer opens an explicit gap the other does not cover.
