# Architecture — document-forge-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ForgeEndpoint` accepts a submission, writes a `ForgeSubmitted` event onto `ForgeEntity`, and immediately starts a `ForgeWorkflow` instance. The workflow's `forgeStep` calls `DocumentForgeAgent` — the single AutonomousAgent — with the prompt and style template as `TaskDef.instructions(...)`. Inside the agent loop, the agent calls the `write_document` tool; the `before-tool-call` guardrail (`WriteDocumentGuardrail`) intercepts that call and validates the path, content, and format before the write lands. Once the tool call passes, the agent returns a `ForgeResult` and the workflow writes `ForgeCompleted` via `ForgeEntity.completeForge`. The `ForgeOutputConsumer` Consumer subscribes to `ForgeCompleted` and runs a parallel quality check, writing `AuditEntry` back via `attachAudit`. The workflow also runs `ForgeAuditor` in `auditStep` — a deterministic scorer that produces an identical `AuditEntry` inline. `ForgeView` projects every entity event into a read-model row; `ForgeEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `ForgeAuditor` and `ForgeOutputConsumer` are both deterministic Java components — they never call a model. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two noteworthy moments:

1. The `before-tool-call` guardrail intercept inside the agent loop — sub-millisecond when the path is valid, but it adds one round-trip to the agent loop when a rejection fires and the agent must retry.
2. The `ForgeOutputConsumer` subscription lag between `ForgeCompleted` and the `attachAudit` call — sub-second in normal operation, running in parallel to `auditStep`.

The agent call itself is bounded by `forgeStep`'s 60 s timeout. The `auditStep` is synchronous and finishes in milliseconds.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → FORGING → FORGE_COMPLETED → AUDITED`.
- Two failure transitions land in `FAILED`: a workflow startup error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `FORGING`. A `FAILED` forge's prior data is preserved on the entity — the UI shows the partial state.
- There is no `APPROVED` state. The generated document is immediately available at `AUDITED`; downstream use is at the requester's discretion.

## Entity model

`ForgeEntity` is the source of truth. It emits five event types. `ForgeView` projects every event into a row used by the UI. `ForgeOutputConsumer` subscribes to entity events to run the quality check. `ForgeWorkflow` both reads (`getForge`) and writes (`markForging`, `completeForge`, `attachAudit`, `fail`) on the entity. The relationship between `DocumentForgeAgent` and `ForgeResult` is "returns" — the agent's task result is the forge result record.

## Defence-in-depth governance flow

For any document that lands in the entity log, the generation passed through:

1. **DocumentForgeAgent** — one model call, one tool invocation, one structured output.
2. **before-tool-call guardrail** — path traversal, oversized content, and disallowed format hints are caught before the write_document tool executes.
3. **On-completion auditor** — every completed document gets a 1–5 quality score so the operator knows which outputs to inspect before use.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
