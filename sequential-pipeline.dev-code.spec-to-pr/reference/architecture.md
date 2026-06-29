# Architecture — spec-to-pr

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `SpecEndpoint` accepts a `{specText}` POST, writes `RunCreated` onto `SpecRunEntity`, and starts `SpecPipelineWorkflow` keyed by `"pipeline-" + runId`. The workflow's first step (`parseStep`) emits `ParseStarted`, then calls `ImplementationAgent` with `TaskDef.taskType(PARSE_SPEC)` and a `phase = PARSE` metadata tag. The agent invokes `ParseTools.extractRequirements` and `ParseTools.identifyAffectedFiles`; every call passes through `WriteGuardrail` first. Once the agent returns a `ParsedSpec`, the workflow writes `SpecParsed` onto the entity and advances to `planStep` — same pattern, the PLAN task carries `phase = PLAN`. Then `draftStep` runs with `phase = DRAFT`. After `PrDrafted` lands, the workflow calls `Workflow.pause()` and the run enters `AWAITING_REVIEW`. A human reviewer approves or rejects via the REST endpoint; the approval signal resumes the workflow, which runs `CiScorer` and writes `CiPassed` or `CiFailed`. `SpecRunView` projects every event into a read-model row; `SpecEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `CiScorer` is a deterministic rule-based scorer; `ReviewHold` is a pure workflow pause. That is what makes this a faithful **single-agent sequential-pipeline** example with HITL and CI governance layered on top.

## Interaction sequence

The sequence traces the happy path: parse → plan → draft → human approval → CI pass → `MERGE_READY`. Three properties are worth pausing on:

1. **The task boundary IS the dependency contract.** Between `parseStep` and `planStep`, the workflow writes `SpecParsed` onto the entity. The next step reads `parsedSpec` from the entity to build the PLAN task's instruction context. The agent never sees parse-phase context inside the plan task's conversation; the typed handoff is the only path information travels.
2. **Every tool call is filtered through `WriteGuardrail`.** The guardrail reads the in-flight task's `phase` metadata and the current `SpecRunEntity.status`, and applies the per-phase accept matrix. A misordered call is rejected before the tool body executes.
3. **ReviewHold is an indefinite pause.** The workflow does not set a timer on `reviewHoldStep`. Only an explicit `ReviewApproved` or `ReviewRejected` signal from a human unblocks it.

## State machine

Fourteen states. The interesting paths:

- The happy path walks `CREATED → PARSING → PARSED → PLANNING → PLANNED → DRAFTING → DRAFTED → AWAITING_REVIEW → CI_RUNNING → CI_PASSED → MERGE_READY`.
- Reviewer rejection lands at `REJECTED` (terminal, no CI run).
- CI failure lands at `CI_FAILED` (terminal, blocked from merge).
- Three failure transitions land in `FAILED`: an agent error during `PARSING`, `PLANNING`, or `DRAFTING`.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no automatic retry from `CI_FAILED` or `REJECTED`. A new run must be submitted — the deployer re-submits the spec after addressing the root cause.

## Entity model

`SpecRunEntity` is the source of truth. It emits fourteen event types — three lifecycle starts, three lifecycle completions, two review decisions, two CI verdicts, the guardrail audit, the failure, and the initial creation, plus `CiStarted`. `SpecRunView` projects every event into a row used by the UI. `SpecPipelineWorkflow` both reads (`getRun`) and writes extensively on the entity. The relationship between `ImplementationAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any run that lands in the entity log, the spec passed through:

1. **Phase-gate guardrail** — every tool call is filtered. A PLAN-phase tool called during PARSE is rejected before the tool body runs; a `GuardrailRejected` event records the violation for audit.
2. **ImplementationAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Human review hold** — the workflow parks after `PrDrafted`. A human reads the diff and explicitly approves or rejects. No auto-advance.
4. **CI gate** — after approval, `CiScorer` runs three deterministic checks. A PR that fails any check cannot reach `MERGE_READY`.

Each layer is independent. The guardrail does not check diff quality; the CI scorer does not check phase order; the human reviewer is not replaced by either. Removing one opens an explicit gap the others do not silently cover.
