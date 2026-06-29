# Architecture — code-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `EditEndpoint` accepts a task submission, writes a `TaskSubmitted` event onto `EditEntity`, and starts an `EditWorkflow` instance. The workflow's `analyzeStep` calls `CodeAssistantAgent` — the single AutonomousAgent — with the task description as `TaskDef.instructions(...)` and each repository file as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`ToolCallGuardrail`) intercepts every tool invocation before execution; allowed calls pass through, disallowed calls return a structured rejection so the agent retries without the blocked call. Once the agent returns an `EditPlan`, the workflow writes `EditProposed` onto the entity. The `CIGate` Consumer subscribes to `EditProposed`, applies the proposed file changes to an in-memory snapshot, runs the test command, and writes `GatePassed` or `GateFailed` back. The workflow's `awaitGateStep` polls the entity until the gate result lands. `EditView` projects every entity event into a read-model row; `EditEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The CI gate (`CIGate`) is a deterministic subprocess runner — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note three distinct points where the system waits:

1. The `analyzeStep` pause while the LLM processes the repository files and generates the edit plan — bounded by the 90 s step timeout to accommodate multi-file analysis.
2. The `CIGate` Consumer processing lag between `EditProposed` and the gate result landing — typically a few seconds for in-process test execution.
3. The `awaitGateStep` polling loop inside the workflow — polls `EditEntity` every 2 s up to its 30 s timeout, advancing as soon as `edit.gateResult().isPresent()` returns true.

The `before-tool-call` guardrail intercepts synchronously inside the agent loop. When a disallowed tool is blocked, the rejection returns to the agent before any execution occurs.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → ANALYZING → EDIT_PROPOSED → GATE_PASSED`.
- The gate-failure path is `SUBMITTED → ANALYZING → EDIT_PROPOSED → GATE_FAILED`. A `GATE_FAILED` card surfaces the test output inline — the developer can inspect what broke and decide whether to request a revised plan.
- Two failure transitions land in `FAILED`: a submit error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `ANALYZING`.
- There is no `APPLIED` or `MERGED` state. The plan is advisory; the developer applies it outside the system. The blueprint deliberately stops at `GATE_PASSED` or `GATE_FAILED`.

## Entity model

`EditEntity` is the source of truth. It emits six event types. `EditView` projects every event into a row used by the UI. `CIGate` subscribes to entity events to run the CI simulation. `EditWorkflow` both reads (`getEdit`) and writes (`markAnalyzing`, `recordPlan`, `recordGateResult`, `fail`) on the entity. The relationship between `CodeAssistantAgent` and `EditPlan` is "returns" — the agent's task result is the plan record.

## Defence-in-depth governance flow

For any edit plan that reaches `GATE_PASSED` in the entity log, the proposal passed through:

1. **ToolCallGuardrail** — every tool call during analysis was on the allowlist; no shell execution or file mutations happened during the agent's reasoning phase.
2. **CodeAssistantAgent** — one model call, one structured output (`EditPlan`).
3. **CIGate** — the proposed changes were applied to an in-memory snapshot and the project's own test suite ran against them; zero failing tests is the gate condition.

Each step is independent. A plan that passes the guardrail can still fail the CI gate. A plan that passes the CI gate is still advisory — the developer reads it and decides.
