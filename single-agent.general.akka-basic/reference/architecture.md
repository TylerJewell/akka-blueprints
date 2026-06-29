# Architecture — akka-basic

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one LLM call. `PromptEndpoint` accepts a run submission, writes the request to `PromptEntity`, and immediately starts a `PromptWorkflow` instance. The workflow's `processStep` calls `PromptAgent` — the single AutonomousAgent — with the prompt text and category hint as `TaskDef.instructions(...)`. The agent's `after-agent-response` guardrail (`OutputGuardrail`) validates each candidate result before it leaves the agent loop. Once a result passes, the workflow writes `ResultRecorded` to the entity. `PromptView` projects every entity event into a read-model row; `PromptEndpoint` serves the read model over REST and SSE.

There is no second agent. `OutputGuardrail` is a deterministic structural validator — it makes no LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). There is one point where the system waits: the `processStep`'s call to the agent, bounded by the 60 s step timeout. Everything before (submit → entity → workflow start) and after (result → entity → view) is synchronous and finishes in milliseconds.

## State machine

Four states. The two paths:

- The happy path is `SUBMITTED → PROCESSING → COMPLETED`.
- The failure path is `SUBMITTED → PROCESSING → FAILED` — entered when the agent exhausts its 3-iteration retry budget without producing a valid result, or when `processStep` hits its 60 s timeout after 2 retries.
- There is no intermediate state between `PROCESSING` and `COMPLETED`. The guardrail's retries are invisible to the entity: `ResultRecorded` only lands once, carrying a valid result.

## Entity model

`PromptEntity` is the source of truth. It emits four event types. `PromptView` projects every event into a row used by the UI. `PromptWorkflow` both reads (`getRun`) and writes (`markProcessing`, `recordResult`, `fail`) on the entity. `PromptAgent` returns a `PromptResult`; the workflow records it via `PromptEntity.recordResult`.

## The single governance check

For any result that lands in the entity log, the candidate passed `OutputGuardrail`'s three checks:

1. `outputText` is non-empty — the model produced a response.
2. `confidenceScore` is in `[0.0, 1.0]` — the value is usable downstream without clamping.
3. `category` is one of the three allowed enum values — callers can switch on it safely.

Each check is independent. A result can pass two and fail one, triggering a targeted retry message naming the failing check. The entity never records a partial result.
