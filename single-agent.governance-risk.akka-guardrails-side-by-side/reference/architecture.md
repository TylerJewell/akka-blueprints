# Architecture — guardrails-side-by-side

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on `PolicyAgent` — the single advisory LLM — flanked by two guardrail agents that gate the pipeline at its entry and exit. `PromptEndpoint` accepts a submission, writes a `PromptReceived` event onto `PromptEntity`, and starts `GuardrailWorkflow`. The workflow's first step calls `InputScreenerAgent` with the user's prompt text. If the screener returns BLOCK, the workflow writes `PromptBlocked` and ends — `PolicyAgent` is never started. If the screener returns PASS, the workflow calls `PolicyAgent` with the prompt and the policy-profile context. Once `PolicyAgent` returns its reply, the workflow calls `OutputValidatorAgent` with the reply. If the validator returns BLOCK, the workflow writes `ResponseBlocked` and ends — the reply is stored on the entity for audit but never surfaced as accepted. If the validator returns PASS, the workflow writes `ResponseAccepted`, runs `ResponseEvaluator` in a final eval step, and writes `EvaluationScored`. `PromptView` projects every entity event into a read-model row; `PromptEndpoint` serves the read model to the UI over REST and SSE.

The two guardrail agents (`InputScreenerAgent` and `OutputValidatorAgent`) are AutonomousAgents with `maxIterationsPerTask(1)`. They make one decision and return. `PolicyAgent` is the sole agent that produces end-user content; the evaluator (`ResponseEvaluator`) is a deterministic rule-based class with no LLM call. This keeps the pattern honest: one agent answers the user's question, and the surrounding components govern it.

## Interaction sequence

The sequence traces the happy path (J1). Two checkpoints bracket the main agent call:

1. **inputScreenStep** — `InputScreenerAgent` evaluates the prompt. This is the `input_guardrails` equivalent: a dedicated agent that runs synchronously before `PolicyAgent` receives the prompt. Bounded by a 30 s timeout.
2. **outputValidateStep** — `OutputValidatorAgent` evaluates `PolicyAgent`'s reply. This is the `output_guardrails` equivalent: a dedicated agent that runs synchronously after `PolicyAgent` returns and before the response is committed as accepted. Also bounded by 30 s.

The main agent call (`agentCallStep`) is bounded by 60 s to accommodate LLM latency. The eval step is synchronous and finishes in milliseconds.

## State machine

Eight states. The key paths:

- **Happy path**: `RECEIVED → SCREENING → AGENT_RUNNING → VALIDATING → RESPONDED`.
- **Input-blocked path**: `RECEIVED → SCREENING → BLOCKED`. `PolicyAgent` is never invoked. The entity records the screener's reason.
- **Output-blocked path**: `RECEIVED → SCREENING → AGENT_RUNNING → VALIDATING → RESPONSE_BLOCKED`. The agent's reply is stored on the entity for audit but flagged as blocked. The caller receives the validator's block reason.
- **Failure path**: a workflow error or agent timeout transitions the entity to `FAILED`. Partial data from the interrupted steps is preserved.

There is no `APPROVED` or `REJECTED` terminal state. Both `RESPONDED` and `BLOCKED`/`RESPONSE_BLOCKED` are terminal. The system never acts on the advice; a human reads the response and decides outside the system.

## Entity model

`PromptEntity` is the source of truth. It emits ten event types covering the full lifecycle, including both block paths. `PromptView` projects every event into a row used by the UI. `GuardrailWorkflow` is the only writer of state-transition events; `PromptEndpoint` writes only `PromptReceived`. The three agent relationships (`InputScreenerAgent → ScreeningVerdict`, `PolicyAgent → PolicyResponse`, `OutputValidatorAgent → ValidationVerdict`) are "returns" — each agent's task result flows back to the workflow and is then written to the entity.

## Guardrail pattern mapping

The OpenAI Agents SDK `input_guardrails` and `output_guardrails` hooks both accept a list of guardrail agents that run alongside the main agent. Akka expresses the same pattern as workflow steps that call purpose-built AutonomousAgents before and after the main agent call:

| SDK concept | Akka equivalent |
|---|---|
| `input_guardrails` — runs before the main agent | `inputScreenStep` calls `InputScreenerAgent` before `agentCallStep` |
| `output_guardrails` — runs after the main agent | `outputValidateStep` calls `OutputValidatorAgent` after `agentCallStep` |
| Guardrail agent returns `GuardrailFunctionOutput(tripwire_triggered)` | `InputScreenerAgent` / `OutputValidatorAgent` return `ScreeningVerdict` / `ValidationVerdict` with `result: BLOCK` |
| Tripwire triggers exception, halts pipeline | Workflow transitions to `blockStep`, entity records block event, caller receives block reason |

The structural difference is that Akka models the pipeline as an explicit workflow with named steps rather than an implicit callback chain. This makes the block points auditable: every `PromptBlocked` or `ResponseBlocked` event carries the structured verdict for the full history.
