# Architecture — sequential-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs four tasks in sequence. `JobEndpoint` accepts a `{jobName, jobType, parameters}` POST, writes `JobCreated` onto `JobEntity`, and starts `JobPipelineWorkflow` keyed by `"pipeline-" + jobId`. The workflow's first step (`validateStep`) emits `ValidateStarted`, then calls `WorkflowAgent` with `TaskDef.taskType(VALIDATE_JOB)` and a `phase = VALIDATE` metadata tag. The agent invokes `ValidateTools.checkFields` and `ValidateTools.verifyConstraints`; every call passes through `StepGuardrail` first. Once the agent returns a `ValidationResult`, the workflow writes `JobValidated` onto the entity and advances to `enrichStep` — same pattern, the ENRICH task carries `phase = ENRICH`. Then `executeStep` runs with `phase = EXECUTE`, and finally `summarizeStep` with `phase = SUMMARIZE`. After `JobSummarized` lands, `evalStep` runs `QualityScorer` over the recorded `(ValidationResult, EnrichedJob, JobOutput, JobSummary)` — no LLM call — and writes `QualityScored`. `JobView` projects every event into a read-model row; `JobEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `QualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `validateStep` and `enrichStep`, the workflow writes `JobValidated` onto the entity. The next step then reads `validation` from the entity to build the ENRICH task's instruction context. The agent never sees validate-phase context inside the enrich task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `StepGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `JobEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes.

The agent calls themselves are bounded by per-step timeouts (60 s on validate / enrich / execute / summarize). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → VALIDATING → VALIDATED → ENRICHING → ENRICHED → EXECUTING → EXECUTED → SUMMARIZING → SUMMARIZED → EVALUATED`.
- Four failure transitions land in `FAILED`: an agent error during `VALIDATING`, `ENRICHING`, `EXECUTING`, or `SUMMARIZING`. A `FAILED` job's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `DISPATCHED` state. The job result is advisory; the operator reviews it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`JobEntity` is the source of truth. It emits twelve event types — four lifecycle starts, four lifecycle completions, the quality evaluation, the guardrail audit, the failure, and the initial creation. `JobView` projects every event into a row used by the UI. `JobPipelineWorkflow` both reads (`getJob`) and writes (`startValidate`, `recordValidation`, `startEnrich`, `recordEnrichment`, `startExecute`, `recordOutput`, `startSummarize`, `recordSummary`, `recordQuality`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `WorkflowAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any job result that lands in the entity log, the job definition passed through:

1. **Phase-gate guardrail** — every tool call is filtered. An EXECUTE-phase tool called during VALIDATE is rejected before the tool body runs; a `GuardrailRejected` event records the violation for audit.
2. **WorkflowAgent (4 task runs)** — four model calls, four structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **On-completion evaluator** — every emitted job result gets a 1–5 quality score. Step coverage, artifact attribution, artifact provenance, and entry parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check artifact completeness; the evaluator does not check phase order. Removing one of them opens an explicit gap the other does not silently cover.
