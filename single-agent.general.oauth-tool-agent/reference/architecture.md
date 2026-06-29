# Architecture — ae-oauth

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SessionEndpoint` accepts a session creation request, writes `SessionCreated` onto `SessionEntity`, and immediately starts a `SessionWorkflow` instance. The workflow's `resolveTokenStep` consults `TokenRegistry` to fetch the `TokenSpec` for the submitted token id, checks for expiry, and writes `TokenResolved` back to the entity (or `SessionFailed` if the token is expired). The workflow then enters `agentStep`, which calls `ToolCallerAgent` — the single AutonomousAgent — with the request text and resolved scope list in the task instructions. As the agent proposes each tool call, `OAuthScopeGuardrail` fires via the `before-tool-call` hook: it looks up the tool's required scope in `ToolRegistry` and either allows the call through to `SimulatedToolExecutor` or returns a structured denial the agent records in its `toolCalls` list. Once the task completes, the workflow writes `ResultRecorded`, calls `complete()` on the entity, and the session reaches `COMPLETED`. `SessionView` projects every entity event into a read-model row; `SessionEndpoint` serves the read model over REST and SSE.

There is no second agent. `OAuthScopeGuardrail`, `TokenRegistry`, `ToolRegistry`, and `SimulatedToolExecutor` are all plain Java classes — the single-agent invariant is preserved.

## Interaction sequence

The sequence traces the happy path (J1) with a `full-access` token. Two distinct enforcement points are visible:

1. **Token resolve**: `resolveTokenStep` calls `TokenRegistry` before the agent is invoked. An expired token exits here with `SessionFailed` — no LLM call is made.
2. **Per-call scope check**: `OAuthScopeGuardrail` fires once per proposed tool call inside the agent loop. A token with partial scope coverage produces a mix of allow and deny responses in the same task execution.

The agent call itself is bounded by `agentStep`'s 90 s timeout. `recordStep` is near-instantaneous — it writes `complete()` to the entity with no external work.

## State machine

Five states. The interesting paths:

- The happy path is `PENDING → TOKEN_RESOLVED → RUNNING → COMPLETED`.
- Two transitions land in `FAILED`: an expired token during `PENDING` (before the agent is invoked) and an agent error during `RUNNING`. In both cases the prior entity state is preserved for audit.
- There is no `APPROVED` or `REJECTED` state for the session as a whole. The per-call disposition lives inside `AgentResult.toolCalls`; the session outcome is `SUCCESS`, `PARTIAL`, or `DENIED` — not a binary session-level gate. The session lifecycle only fails if the infrastructure breaks, not because the agent was denied scope.

## Entity model

`SessionEntity` is the source of truth. It emits five event types. `SessionView` projects every event into a row for the UI. `SessionWorkflow` both reads (`getSession`) and writes (`resolveToken`, `markRunning`, `recordResult`, `complete`, `fail`) on the entity. `ToolCallerAgent` has a one-directional relationship: it returns `AgentResult` to the workflow; it does not write to the entity directly. `OAuthScopeGuardrail` consults `ToolRegistry` on every proposed call but has no entity dependency.

## Scope enforcement flow

For any `AgentResult` that lands in the entity log, every tool call in its `toolCalls` list passed through:

1. **Token resolve** — the token id was valid and not expired before the agent was invoked.
2. **ToolCallerAgent** — one model call, with multi-turn tool-call rounds inside a single task.
3. **OAuthScopeGuardrail** — every proposed tool call was checked against the resolved scope set before execution. Denials are recorded in the result, not silently dropped.

The scope check is structurally impossible to bypass: the guardrail is wired inside the agent loop via the `before-tool-call` hook, not after the agent returns.
