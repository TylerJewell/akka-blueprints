# Architecture — tool-search-discovery

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `TaskEndpoint` accepts a task submission, writes a `TaskReceived` event onto `TaskEntity`, and immediately starts a `DiscoveryWorkflow` instance. The workflow's `discoverStep` queries `ToolRegistry` — an in-process map seeded from `tool-catalog.jsonl` and kept current by `ToolCatalogConsumer` — for tools relevant to the user query. The discovered schemas are written back to `TaskEntity` via `recordDiscovered` and then bundled into a catalog attachment. The workflow's `executeStep` calls `DiscoveryAgent` — the single AutonomousAgent — with the user query as `TaskDef.instructions(...)` and the catalog snapshot as a `TaskDef.attachment(...)`. The agent's `before-tool-invocation` guardrail (`ToolAllowlistGuardrail`) validates every outbound tool call before it executes. Once the agent returns a `TaskResult`, the workflow writes `TaskCompleted`. `TaskView` projects every entity event into a read-model row; `TaskEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `ToolRegistry.search()` is a deterministic in-process lookup — not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two distinct moments where the system does work before the agent call:

1. `DiscoveryWorkflow.discoverStep` — a synchronous registry query that runs in milliseconds. The catalog snapshot it produces becomes the agent's context.
2. `DiscoveryWorkflow.executeStep` — the LLM call itself, bounded by a 60 s step timeout.

The guardrail intercept is synchronous within the agent loop. A blocked tool call returns to the agent immediately; the agent adjusts its plan and may attempt a different tool in the same iteration without advancing the iteration counter.

## State machine

Six states. The interesting paths:

- The happy path is `RECEIVED → DISCOVERING → READY → EXECUTING → COMPLETED`.
- Two `FAILED` transitions: a registry error during `DISCOVERING`, and an agent error (or all tools blocked by the guardrail) during `EXECUTING`. A `FAILED` task's prior data — including the discovered tool list — is preserved on the entity for debugging.
- `COMPLETED` tasks record both `toolsUsed` and `guardedTools`. A task that completed with some tools blocked is still `COMPLETED`, not `FAILED`, as long as the agent produced a non-empty output.

## Entity model

`TaskEntity` is the source of truth. It emits five event types. `TaskView` projects every event into a row for the UI. `ToolCatalogConsumer` updates `ToolRegistry` independently of `TaskEntity`. `DiscoveryWorkflow` both reads (`getTask`) and writes (`recordDiscovered`, `markExecuting`, `completeTask`, `fail`) on the entity. `DiscoveryAgent`'s relationship to `TaskResult` is "returns" — the agent's task output is the result record.

## Defence-in-depth governance flow

For any task that lands in `COMPLETED` state, the execution path passed through:

1. **ToolRegistry search** — the agent only sees schemas for tools that exist in the catalog; it cannot call a tool with no schema.
2. **DiscoveryAgent** — one model call, one structured output.
3. **before-tool-invocation guardrail** — every tool invocation is checked against the allowlist before it executes; blocked ids land in `TaskResult.guardedTools` for operator review.

Each step is independent. Removing the guardrail opens an explicit gap: the agent could invoke catalog tools that the operator has not approved for this deployment.
