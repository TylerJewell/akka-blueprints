# Architecture — hook-instrumentation

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call with a three-layer hook chain around it. `ObservationEndpoint` accepts a task submission and writes a `TaskSubmitted` event onto `ObservationEntity`. The `HookLogConsumer` Consumer subscribes, builds the `HookConfig` from the task's tool registry and seeded blocked-tool lists, writes it back via `initHookChain`, and starts an `ObservationWorkflow` instance.

The workflow's `hookInitStep` confirms the entity is ready, then `executeStep` calls `ActivityObserverAgent` — the single AutonomousAgent — with the task description and available tool list. Inside the agent's loop, three hook observers fire at distinct activity boundaries:

- `BeforeToolCallGuardrail` fires before every tool invocation, blocking disallowed tools and validating parameter schemas.
- `AfterToolCallGuardrail` fires after every tool returns, scanning the output for sensitive patterns and redacting matches before they enter the agent's next reasoning step.
- `BeforeLlmCallGuardrail` fires before every LLM invocation, stripping credential-like tokens from the assembled prompt.

The guardrails accumulate `HookLogEntry` records into shared task context; the agent's final `AgentOutcome` carries the full hook log. Once the outcome lands, the workflow writes `OutcomeRecorded` and runs `CoverageScorer` in `coverageStep`. The coverage score lands as `CoverageScored`. `ObservationView` projects every entity event into a read-model row; `ObservationEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. The coverage scorer (`CoverageScorer`) is a deterministic rule-based checker — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct hook-fire moments are visible:

1. `BeforeLlmCallGuardrail` fires before the first LLM invocation, typically recording `PASS_THROUGH` on a clean task.
2. `BeforeToolCallGuardrail` fires for each tool the agent attempts, recording `ALLOWED` for permitted tools and `BLOCKED` for any in the blocked list.
3. `AfterToolCallGuardrail` fires after each successful tool return, recording `PASS_THROUGH` when no sensitive patterns match or `REDACTED` when they do.

The agent call is bounded by `executeStep`'s 90 s timeout to accommodate multi-tool task latency. The `coverageStep` is synchronous and finishes in milliseconds.

## State machine

Five states. The notable paths:

- The happy path is `SUBMITTED → HOOK_CHAIN_READY → EXECUTING → COMPLETED`.
- Two failure transitions land in `FAILED`: a hook init error during `SUBMITTED`, and an agent error (or all-iterations-exhausted) during `EXECUTING`. A `FAILED` observation's prior data is preserved on the entity — the UI shows the partial hook log for the operator.
- There is no `APPROVED` state. This blueprint terminates at `COMPLETED`. Action taken on the task outcome is out of scope; the hook log is the governance artifact.

## Entity model

`ObservationEntity` is the source of truth. It emits six event types. `ObservationView` projects every event into a row used by the UI. `HookLogConsumer` subscribes to entity events to initialize the hook chain and trigger the workflow. `ObservationWorkflow` both reads (`getObservation`) and writes (`markExecuting`, `recordOutcome`, `recordCoverage`, `fail`) on the entity. `ActivityObserverAgent`'s relationship to `AgentOutcome` is "returns" — the task result is the outcome record, which includes the accumulated hook log from all three guardrails.

## Defence-in-depth governance flow

For any outcome that lands in the entity log, the agent's activity passed through:

1. **Before-tool-call guardrail** — disallowed tools are blocked before any side effect; only the blocked tool name and input hash are recorded, not the tool's result.
2. **ActivityObserverAgent** — one model loop, producing tool calls and a final response.
3. **After-tool-call guardrail** — sensitive patterns in tool outputs are redacted before re-entering the agent's context; the original output hash is in the audit log.
4. **Before-llm-call guardrail** — credential-like tokens are stripped from every assembled prompt; the model never receives them.
5. **Coverage scorer** — every completed outcome gets a 1–5 completeness score so the operator knows at a glance whether every hook fired as expected.

Each layer is independent. A misconfigured or missing hook creates an explicit visible gap in the coverage score rather than a silent pass.
