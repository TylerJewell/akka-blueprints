# Architecture — skill-patterns-tutorial

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `SkillRunEndpoint` accepts a skill invocation request, writes it to `SkillRunEntity`, and starts a `SkillRunWorkflow` instance. The workflow's `runStep` calls `SkillDemoAgent` — the single AutonomousAgent — with the pattern and input encoded as task text.

Inside the agent, the dispatch to a specific skill happens based on the `pattern` field. For the INLINE pattern the skill's prompt never leaves the agent class; for the FILE_BASED pattern the prompt was loaded from a classpath resource at startup; for the EXTERNAL pattern the agent invokes `SkillToolStub` via an HTTP tool call and incorporates the response; for the META_CREATOR pattern the agent synthesizes a `SkillDefinition` JSON block in its reasoning output.

The workflow's `recordStep` writes `SkillRunCompleted` back to the entity. `SkillRunView` projects all entity events into a row used by the list on the UI. `SkillRunEndpoint` serves the view over REST and SSE.

`SkillToolStub` is a plain `HttpEndpoint` — not an agent. It returns canned key-value data and exists entirely to give the external-skill pattern a real outbound tool call to demonstrate, without requiring any deployed third-party service.

## Interaction sequence

The sequence traces the happy path for any pattern (J1). Two moments to note:

1. For the EXTERNAL pattern, there is an inner tool-call round trip between `SkillDemoAgent` and `SkillToolStub` before the agent produces its `SkillResult`. This happens inside the agent's single task execution and is bounded by `runStep`'s 90 s timeout.
2. The `recordStep` is short (10 s timeout) — it is a single entity command call that appends `SkillRunCompleted`.

For the META_CREATOR pattern the agent's reasoning is more open-ended; the 90 s `runStep` timeout accommodates the additional token generation needed to produce a well-formed `SkillDefinition` JSON block.

## State machine

Four states. The happy path is `REQUESTED → RUNNING → COMPLETED`. The only failure path is `RUNNING → FAILED`, triggered when the agent times out or returns a response that cannot be deserialized into `SkillResult`. A `FAILED` run's request is preserved on the entity; the UI shows the partial state.

There is no retry loop at the entity level. The workflow's `defaultStepRecovery maxRetries(1)` gives `runStep` one agent-level retry before failing over to `errorStep`. This is intentional: a skill-pattern tutorial that silently retried indefinitely would obscure the failure mode a developer needs to see.

## Entity model

`SkillRunEntity` is the source of truth. It emits four event types. `SkillRunView` projects every event into a row. `SkillRunWorkflow` both reads (`getRun`) and writes (`markRunning`, `recordResult`, `fail`) on the entity. `SkillDemoAgent`'s relationship to `SkillResult` is "returns" — the agent task result is the typed record stored on the entity.

`SkillToolStub`'s relationship to `SkillDemoAgent` is a tool-call channel, not a component-client call. The agent calls the stub's HTTP endpoint via a `ToolDefinition.http(...)` registered in its skill definition; the stub has no knowledge of which agent invoked it.

## Skill-wiring summary

| Pattern | Wiring point | Load time | Recompile needed to change? |
|---|---|---|---|
| Inline | `Skill.inline(...)` in `SkillDemoAgent.definition()` | Startup | Yes |
| File-based | `Skill.fromResource(...)` pointing to classpath resource | Startup | No — swap the file |
| External | `Skill.withTool(...)` + `ToolDefinition.http(...)` | Startup | No — change the URL or tool response |
| Meta creator | Agent reasoning at inference time | Per-request | Not applicable — the skill is generated, not pre-loaded |

This table is what the developer reads when deciding which pattern fits their use case.
