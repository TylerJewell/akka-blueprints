# Architecture — inline-agent-demo

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `RunEndpoint` accepts a submission, writes a `RunReceived` event onto `RunEntity`, and starts a `RunWorkflow`. The workflow's `validateStep` reads the entity to check that the inline agent definition is well-formed. If it is not, the workflow calls `RunEntity.fail(reason)` and stops — no LLM is invoked. On a valid definition, the workflow's `runStep` marks the run as started and calls `InlineAgentRunner` — the single AutonomousAgent — constructing its `AgentDefinition` at call time from the caller-supplied `systemPrompt`, `outputSchema`, and `allowedTools`. The agent returns an `AgentResponse`; the workflow writes `RunCompleted` to the entity. `RunView` projects every entity event into a read-model row; `RunEndpoint` serves the read model to the UI over REST and SSE.

The graph has no guardrail, no sanitizer, and no evaluator. This is deliberate: the inline-agent demo is a structural showcase. Deployers who fork the blueprint add controls appropriate to their use case.

## Interaction sequence

The sequence traces the happy path (J1). The critical observation is where the agent definition enters the system:

1. The caller POSTs an `InlineAgentRequest` including the full `AgentDefinition` — this is the moment the definition is recorded.
2. The workflow's `runStep` reads the stored definition from the entity and constructs the `AgentDefinition` object passed to `InlineAgentRunner`. The base system prompt (from `prompts/inline-agent-runner.md`, loaded at startup) is prepended.
3. The agent runs to completion. No guardrail wraps the response — the caller owns the output schema contract.

## State machine

Four states. Two terminal states — `COMPLETED` and `FAILED` — can both be reached from two different points:

- A validation error during `RECEIVED` sends the run straight to `FAILED` without ever starting an LLM call.
- An agent error (task timeout, model error, or exhausted retries) during `RUNNING` sends the run to `FAILED` after the LLM call was attempted.

There is no intermediate holding state between `RECEIVED` and `RUNNING` — the workflow's `validateStep` is synchronous and fast.

## Entity model

`RunEntity` is the source of truth. It emits four event types. `RunView` projects every event into a row used by the UI. `RunWorkflow` both reads (`getRun`) and writes (`markRunning`, `complete`, `fail`) on the entity. `InlineAgentRunner` is related to `AgentResponse` by a "returns" relationship — the agent's task result is the response record.

## Inline agent construction

The defining feature of this blueprint is that `InlineAgentRunner` does not have a fixed system prompt baked in at class definition time. Instead, the `AgentDefinition` object is constructed inside `runStep` by:

1. Loading the base instructions from `prompts/inline-agent-runner.md` (done at startup, cached).
2. Appending the caller-supplied `agentDefinition.systemPrompt`.
3. Registering the caller-supplied `agentDefinition.allowedTools` as the tool set for this task.
4. Setting the output shape expectation from `agentDefinition.outputSchema`.

This means a single `InlineAgentRunner` class can serve as a Q&A assistant, a keyword extractor, a tone classifier, or any other stateless task — without a redeployment.
