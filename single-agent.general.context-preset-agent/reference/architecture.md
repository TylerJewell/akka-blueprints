# Architecture — context-preset-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `PresetRequestEndpoint` accepts a submission, writes a `RequestSubmitted` event onto `PresetRequestEntity`, and starts a `PresetRequestWorkflow` instance. The workflow's `resolvePresetStep` reads the matching `PresetDefinition` from `PresetRegistry` — a Key-Value Entity keyed by `"<env>:<role>"`. Once resolved, the preset is attached to the entity via `resolvePreset` and the workflow advances to `executeStep`. The `executeStep` calls `ContextPresetAgent` — the single AutonomousAgent — with the preset's `instructionAddendum` and the caller's `requestText` as task instructions. The agent's `before-tool-call` guardrail (`ToolGatingGuardrail`) intercepts every tool invocation and compares the tool name against `PresetDefinition.allowedTools`. Tool calls within the allowed list proceed; others are rejected with a structured permission-denied message. Once the agent completes its task, `executeStep` writes `RequestCompleted` to the entity. The `auditStep` then tallies INVOKED vs BLOCKED entries and writes `RequestAudited`. `PresetRequestView` projects every entity event into a read-model row; `PresetRequestEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The audit computation in `auditStep` is arithmetic over the completed result's `toolCallLog` — not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1: `prod` + `admin` caller requesting a cache flush). Note the preset resolution step before the agent call — it is synchronous and typically sub-second because `PresetRegistry` is a Key-Value Entity backed by the same Akka runtime.

The guardrail check is inline on every tool call. For `prod:admin` the check passes for `adminActionTool`. The entire `executeStep` is bounded by a 60 s timeout to accommodate LLM latency. The `auditStep` is synchronous arithmetic — milliseconds.

## State machine

Five primary states. The interesting paths:

- The happy path is `SUBMITTED → PRESET_RESOLVED → EXECUTING → COMPLETED`. The `RequestAudited` event lands the entity back in `COMPLETED` (the audit is an additive event on an already-terminal state, not a separate terminal).
- Two failure transitions land in `FAILED`: a missing preset key during `SUBMITTED` (resolvePresetStep cannot find `"prod:admin"` because `PresetRegistry` was not seeded), and an agent error during `EXECUTING` (timeout, model error, or all 3 iterations exhausted). A `FAILED` request's prior data is preserved on the entity for diagnosis.
- There is no rollback of an already-executed tool call. If `adminActionTool` fired before the agent returned and then the workflow failed, the action has already occurred. Deployers with transactional requirements should add a compensation step.

## Entity model

`PresetRequestEntity` is the source of truth. It emits six event types. `PresetRequestView` projects every event into a row used by the UI. `PresetRequestWorkflow` both reads (`getRequest`) and writes (`resolvePreset`, `markExecuting`, `complete`, `recordAudit`, `fail`) on the entity. `PresetRegistry` is a read-only dependency of the workflow — it is never modified by the request lifecycle. `ContextPresetAgent` returns a `PresetRequestResult` that the workflow writes to the entity via `complete`.

## Governance flow

For any completed request in the entity log, the call passed through:

1. **CI configuration gate** — the preset that guided the agent was validated at build time; the artifact cannot contain a malformed preset definition.
2. **PresetRegistry resolution** — the exact `PresetDefinition` (model, allowed tools, instruction addendum) is resolved from stored state before the agent call, not inferred from the request body.
3. **ContextPresetAgent** — one model call, one structured output.
4. **before-tool-call guardrail** — every tool invocation is checked against the resolved `allowedTools` list before the tool executes. Blocks are recorded in the result, never silently dropped.
5. **Audit step** — the completed result's tool call log is tallied; the entity records how many calls were allowed vs blocked for every request.

Each step is independent. Removing the guardrail allows the model to reach any registered tool regardless of role. Removing the CI gate allows a misconfigured preset to reach the runtime without prior validation.
