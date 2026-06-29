# Architecture — react-math-wiki

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ResearchEndpoint` accepts a question, writes a `QuestionSubmitted` event onto `ResearchEntity`, and immediately starts a `ResearchWorkflow` instance. The workflow's `runStep` calls `ResearchAgent` — the single AutonomousAgent — with the question text as `TaskDef.instructions(...)`. Inside the ReAct loop, the agent interleaves calls to `evaluate_math` and `search_wikipedia`. Each `evaluate_math` call is intercepted by `MathGuardrail` before it reaches `MathEvaluator`; `search_wikipedia` calls pass directly to `WikipediaStub`. Every tool invocation emits a `ToolCallRecorded` event on the entity, so the trace is durable and visible in the UI in real time. Once the agent finalises its answer, the workflow writes `AnswerRecorded` and runs `AnswerEvaluator` in `evalStep`. The score lands as `EvaluationScored`. `ResearchView` projects every entity event into a read-model row; `ResearchEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `AnswerEvaluator` is a deterministic rule-based scorer — not an LLM. That is what makes this a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1) for a numeric word-problem. The key moments:

1. `ResearchEndpoint` submits the entity command and starts the workflow in the same request handler — there is no background consumer to wait for.
2. `runStep`'s 90 s timeout accommodates multi-step ReAct loops with real LLM latency. Each ReAct step records a `ToolCallRecorded` event, making the trace observable before the answer arrives.
3. `evalStep` is synchronous and runs in milliseconds — no external service, no LLM call.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → RUNNING → ANSWERED → EVALUATED`.
- `RUNNING` self-loops on every `ToolCallRecorded` event — the entity stays in `RUNNING` throughout the entire ReAct loop, accumulating tool-call trace entries.
- Two failure transitions land in `FAILED`: a workflow-start error during `SUBMITTED`, and an agent error (or guardrail-budget exhaustion) during `RUNNING`. A `FAILED` question's prior trace is preserved on the entity for debugging.
- There is no `APPROVED` state. The answer is informational; no human gate sits between `EVALUATED` and the user reading it.

## Entity model

`ResearchEntity` is the source of truth. It emits six event types. `ResearchView` projects every event into a row used by the UI. `ResearchWorkflow` both reads (`getQuestion`) and writes (`markRunning`, `recordToolCall`, `recordAnswer`, `recordEvaluation`, `fail`) on the entity. The relationship between `ResearchAgent` and `ResearchAnswer` is "returns" — the agent's task result is the answer record.

## Defence-in-depth governance flow

For any answer that lands in the entity log, the question passed through:

1. **ResearchAgent ReAct loop** — one model, up to 8 steps, two tools.
2. **MathGuardrail before-tool-call** — every `evaluate_math` argument is grammar-checked before execution; forbidden constructs are rejected with a structured error, and the agent may reformulate.
3. **MathEvaluator belt-and-suspenders** — the evaluator enforces the same grammar at runtime even if the guardrail were bypassed.
4. **On-decision evaluator** — every answer gets a 1–5 completeness score so the user can see at a glance whether the agent's reasoning chain is well-supported.

Each step is independent. Removing the guardrail opens the math-execution path to arbitrary expressions without the evaluator providing full coverage.
