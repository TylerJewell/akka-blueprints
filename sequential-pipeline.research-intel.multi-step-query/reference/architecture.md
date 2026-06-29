# Architecture — multi-step-query-engine

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `QueryEndpoint` accepts a `{question}` POST, writes `QueryCreated` onto `QueryEntity`, and starts `QueryPipelineWorkflow` keyed by `"pipeline-" + queryId`. The workflow's first step (`decomposeStep`) emits `DecomposeStarted`, then calls `QueryAgent` with `TaskDef.taskType(DECOMPOSE_QUESTION)` and a `phase = DECOMPOSE` metadata tag. The agent invokes `DecomposeTools.generateSubQuestions` and `DecomposeTools.rankSubQuestions`; once it returns a `DecomposedQuestion`, the workflow writes `QuestionDecomposed` onto the entity and advances to `retrieveStep`. The RETRIEVE task carries the recorded sub-questions as instruction context and calls `RetrieveTools.searchPassages` and `RetrieveTools.fetchPassage` per sub-question. After `EvidenceRetrieved` lands, `synthesizeStep` runs — the agent builds sections with `SynthesizeTools.draftSection` and assembles the final record with `SynthesizeTools.composeAnswer`. After `AnswerSynthesized` lands, `evalStep` runs `StoppingEvaluator` over the recorded `(DecomposedQuestion, EvidenceSet, QueryAnswer)` triple — no LLM call — and writes `AnswerEvaluated`. `QueryView` projects every event into a read-model row; `QueryEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `StoppingEvaluator` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `decomposeStep` and `retrieveStep`, the workflow writes `QuestionDecomposed` onto the entity. The next step reads `decomposition` from the entity to build the RETRIEVE task's instruction context. The agent never sees decompose-phase context inside the retrieve task's conversation; the typed handoff is the only path information travels.
2. The retrieve step calls `searchPassages` once per sub-question, then selectively calls `fetchPassage` to enrich specific results. This tool-call pattern is not hardwired — the agent drives it — but the on-decision evaluator penalises any answer section whose `citedPassageIds` reference a passage not present in the returned `EvidenceSet`.

The agent calls are bounded by per-step timeouts (60 s on decompose / retrieve / synthesize). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → DECOMPOSING → DECOMPOSED → RETRIEVING → RETRIEVED → SYNTHESIZING → SYNTHESIZED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `DECOMPOSING`, `RETRIEVING`, or `SYNTHESIZING`. A `FAILED` query's prior data is preserved on the entity — the UI shows the partial state.
- `AnswerEvaluated` always fires; a score of 1 is still a terminal `EVALUATED` state, not a failure. The reader decides whether to re-run the query with a richer question.

There is no `APPROVED` or `PUBLISHED` state. The answer is advisory; the researcher consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`QueryEntity` is the source of truth. It emits nine event types — three lifecycle starts, three lifecycle completions, the evaluation, the failure, and the initial creation. `QueryView` projects every event into a row used by the UI. `QueryPipelineWorkflow` both reads (`getQuery`) and writes (`startDecompose`, `recordDecomposition`, `startRetrieve`, `recordEvidence`, `startSynthesize`, `recordAnswer`, `recordEvaluation`, `fail`) on the entity. The relationship between `QueryAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Governance flow

For any answer that lands in the entity log, the question passed through:

1. **QueryAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase. The agent is stateless across phases; the workflow is the only connector.
2. **On-decision evaluator** — every emitted answer gets a 1–5 quality score. Sub-question coverage, citation provenance, confidence calibration, and section parity are each worth one point on a base of 1.

The evaluator fires unconditionally after every answer. Removing it removes the only automated quality signal; the reader would have no fast indication of which answers are evidence-thin.
