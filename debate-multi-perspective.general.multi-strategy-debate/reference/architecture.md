# Architecture — multi-strategy-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`QueryEndpoint` is the entry point. It writes a `QueryReceived` event to `QueryQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `QueryWorkflow` per submission. The workflow validates the query, then calls five agents — `StrategyCoordinator` to decompose the question, then `KeywordSearchAgent`, `SemanticRetrievalAgent`, and `ChainOfThoughtAgent` in parallel, then `StrategyCoordinator` again to synthesize. Each transition emits an event on `QueryEntity`. `QueryView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `QuerySimulator` drips sample questions for demonstration, and `EvalSampler` ticks every 5 minutes to grade one synthesized query for cross-strategy agreement, calling `ConsistencyJudge`.

The query validator is not a component box — it is the deterministic `QueryValidator` helper invoked inside `QueryWorkflow.validateStep`. It makes no Akka call; it checks the raw question string and either proceeds or emits a rejection.

## Interaction sequence

The sequence diagram traces the happy path (J1). Two notes matter. First, the validate step runs before the coordinator is called and before any agent is involved — a rejected query ends the workflow immediately. Second, the `par` block: the three strategy agents run **concurrently**, not in a chain. The workflow awaits all three via a CompletionStage zip with a 60-second per-step timeout, which is the heart of the debate-multi-perspective pattern. The diversity in retrieval method — exact term match, semantic similarity, and step-by-step inference — is what makes the parallel outputs worth synthesizing rather than simply routing to one strategy.

## State machine

The query has six states. `RECEIVED` is the initial state when `QueryCreated` fires. `REJECTED` is a terminal state reached if the input guardrail fires — no agent is ever called. `RUNNING` is reached once the coordinator has decomposed the question and the three strategy agents have been dispatched. The query then lands in one of three terminal states: `SYNTHESIZED` (all strategies returned and the output guardrail passed), `DEGRADED` (a strategy agent timed out; synthesis ran on partial input), or `BLOCKED` (the output guardrail rejected the synthesized answer). An `AgreementScored` event may land on a `SYNTHESIZED` query without changing its status.

## Entity model

`QueryEntity` is the system's source of truth; every transition writes one of ten event types. `QueryQueue` is the audit log of submitted questions. `QueryView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for each strategy call and for the synthesis call. The default 5-second workflow step timeout (Lesson 4) is far too short for LLM-driven retrieval or reasoning.
- Degraded path: a strategy agent timeout transitions to synthesis-from-partial rather than failing the whole workflow; `failureReason` names the missing strategy.
- Idempotency: `(question, submittedBy)` over a 10 s window deduplicates `POST /api/queries`.
- View indexing: `getAllQueries` has no `WHERE status` clause — the `QueryStatus` enum cannot be auto-indexed (Lesson 2); callers filter client-side.
- Eval sampler: one query per 5-minute tick; the oldest `SYNTHESIZED` query without an `agreementScore` wins.
- Validation ordering: the input guardrail runs before decomposition, which ensures no agent call is charged against a query that will be rejected.
