# Architecture — akka-bridge

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `GraphEndpoint` accepts a `{graphId}` POST, writes `GraphRunCreated` onto `GraphRunEntity`, and starts `GraphExecutionWorkflow` keyed by `"run-" + runId`. The workflow's first step (`planStep`) emits `PlanStarted`, then calls `GraphAgent` with `TaskDef.taskType(PLAN_GRAPH)` and a `phase = PLAN` metadata tag. The agent invokes `PlanTools.parseGraphDefinition` and `PlanTools.estimateNodeCost`; every LLM response passes through `OutputGuardrail` before it is accepted. Once the agent returns an `ExecutionPlan`, the workflow writes `GraphPlanned` onto the entity and advances to `executeStep` — same pattern, the EXECUTE task carries `phase = EXECUTE`. Then `finalizeStep` runs with `phase = FINALIZE`. After `OutputFinalized` lands, `evalStep` runs `PolicyChecker` over the recorded `(ExecutionPlan, NodeOutputSet, FinalResult)` triple — no LLM call — and writes `EvaluationScored`. `GraphRunView` projects every event into a read-model row; `GraphEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `PolicyChecker` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `planStep` and `executeStep`, the workflow writes `GraphPlanned` onto the entity. The next step reads `plan` from the entity to build the EXECUTE task's instruction context. The agent never sees plan-phase context inside the execute task's conversation; the typed handoff is the only path information travels.
2. Every LLM response is filtered through `OutputGuardrail`. The guardrail inspects the raw response for schema conformance, prohibited-content patterns, and reference provenance before the output is committed to the entity or forwarded downstream. A policy-violating response is blocked before it is written.

The agent calls themselves are bounded by per-step timeouts (60 s on plan / execute / finalize). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → PLANNING → PLANNED → EXECUTING → EXECUTED → FINALIZING → FINALIZED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `PLANNING`, `EXECUTING`, or `FINALIZING`. A `FAILED` run's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailViolated` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `DEPLOYED` state. The final result is advisory; the reviewer consumes it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`GraphRunEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `GraphRunView` projects every event into a row used by the UI. `GraphExecutionWorkflow` both reads (`getRun`) and writes (`startPlan`, `recordPlan`, `startExecute`, `recordNodeOutputs`, `startFinalize`, `recordResult`, `recordEvaluation`, `recordViolation`, `fail`) on the entity. The relationship between `GraphAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any run that lands in the entity log, every node output passed through:

1. **After-LLM-response guardrail** — every LLM response is filtered before it is committed. A policy-violating output is blocked; a `GuardrailViolated` event records the violation for audit. The block fires before the output reaches the entity or any downstream node.
2. **GraphAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-decision evaluator** — every finalized output gets a 1–5 coverage-and-provenance score. Node coverage, output integrity, reference provenance, and output parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check coverage; the evaluator does not check prohibited-content patterns. Removing one of them opens an explicit gap the other does not silently cover.
