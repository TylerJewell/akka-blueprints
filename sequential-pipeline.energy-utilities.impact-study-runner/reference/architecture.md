# Architecture — impact-study-runner

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `StudyEndpoint` accepts a `{requestId}` POST, writes `StudyCreated` onto `StudyEntity`, and starts `StudyPipelineWorkflow` keyed by `"pipeline-" + studyId`. The workflow's first step (`loadFlowStep`) emits `LoadFlowStarted`, then calls `StudyAgent` with `TaskDef.taskType(RUN_LOAD_FLOW)` and a `phase = LOAD_FLOW` metadata tag. The agent invokes `LoadFlowTools.computeBusVoltages` and `LoadFlowTools.computePowerFlows`; once it returns a `LoadFlowResult`, the workflow writes `LoadFlowComputed` onto the entity and advances to `contingencyStep` — same pattern, the CONTINGENCY task carries `phase = CONTINGENCY`. Then `draftStep` runs with `phase = DRAFT`. After `ReportDrafted` lands, `evalStep` runs `SimulatorScorer` over the recorded `(LoadFlowResult, ContingencyResult, StudyReport)` triple — no LLM call — and writes `EvaluationScored`. The workflow then pauses in `approvalStep`, emitting `ApprovalRequested` and waiting for an engineer to call the approve or reject endpoint. `StudyView` projects every event into a read-model row; `StudyEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `SimulatorScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path plus the engineer approval (J1 + J2). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `loadFlowStep` and `contingencyStep`, the workflow writes `LoadFlowComputed` onto the entity. The next step reads `loadFlow` from the entity to build the CONTINGENCY task's instruction context. The agent never sees load-flow context inside the contingency task's conversation; the typed handoff is the only path information travels.
2. The approval gate is structural. `approvalStep` does not set a timer; it waits indefinitely for an external signal routed through `StudyEntity.approve` or `StudyEntity.reject`. The eval score is surfaced on the approval card precisely so the engineer deciding has the automated findings in view.

The agent calls are bounded by per-step timeouts (60 s on load-flow / contingency / draft). `evalStep` is synchronous and finishes in milliseconds. `approvalStep` has no timeout.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → RUNNING_LOAD_FLOW → LOAD_FLOW_DONE → RUNNING_CONTINGENCY → CONTINGENCY_DONE → DRAFTING → REPORT_DRAFTED → AWAITING_APPROVAL → APPROVED`.
- Rejection walks the final transition to `REJECTED` instead.
- Three failure transitions land in `FAILED`: an agent error during `RUNNING_LOAD_FLOW`, `RUNNING_CONTINGENCY`, or `DRAFTING`. A `FAILED` study's prior data is preserved on the entity — the UI shows the partial state.
- `AWAITING_APPROVAL` is the only state where the system actively waits for a human. All states before it are driven by the workflow; all states after it are terminal.

There is no auto-approval or auto-rejection. The study stays in `AWAITING_APPROVAL` until an engineer acts.

## Entity model

`StudyEntity` is the source of truth. It emits twelve event types — three lifecycle starts, three lifecycle completions, the evaluation, the approval request, two approval outcomes, the failure, and the initial creation. `StudyView` projects every event into a row used by the UI. `StudyPipelineWorkflow` both reads and writes on the entity: it reads via `getStudy` to build task instruction contexts, and writes via `startLoadFlow`, `recordLoadFlow`, `startContingency`, `recordContingency`, `startDraft`, `recordReport`, `recordEvaluation`, `requestApproval`. The engineer's actions (`approve`, `reject`) arrive via `StudyEndpoint`, not the workflow, which is why the approval gate is structurally separate from the pipeline steps.

## Defence-in-depth governance flow

For any study that reaches the approval gate, its topic passed through:

1. **StudyAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase. The agent is stateless across phases; information only travels through typed entity writes.
2. **On-decision evaluator** — every drafted report gets a 1–5 score. Violation coverage, voltage bounds, N-1 criterion parity, and section count parity are each worth one point on a base of 1. The score and rationale are displayed on the approval card.
3. **Engineer approval gate** — the workflow is paused; no study report is issued without an explicit engineer command. The gate is enforced structurally: the workflow has no auto-transition out of `AWAITING_APPROVAL`.

Each layer is independent. The evaluator does not block the approval gate (it is non-blocking); the approval gate does not check the eval score (the engineer can approve despite a low score, carrying the review burden). Removing one of them opens an explicit gap the other does not silently cover.
