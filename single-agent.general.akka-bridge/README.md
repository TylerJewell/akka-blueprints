# Akka Sample: Agents

Wraps external agent framework invocations inside Akka workflows so every tool call and LLM invocation gains durable execution, automatic retries, and a structured audit trail. The agent framework runs as a tool the workflow calls; Akka owns the lifecycle.

Demonstrates the **single-agent** coordination pattern with one governance mechanism: a `before-tool-call` guardrail that intercepts each outbound tool request before it leaves the Akka runtime, validates it against a per-tool policy, and either permits or blocks it with a structured rejection that the agent loop can act on.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the external agent framework is simulated in-process and all tool calls are handled by seeded mock tool handlers.

## Generate the system

```sh
cp -r ./single-agent.general.akka-bridge  ~/my-projects/akka-bridge
cd ~/my-projects/akka-bridge
```

(Optional) Edit `SPEC.md` to point at a real agent framework invocation by replacing the in-process `AgentFrameworkAdapter` with a live HTTP call to your running agent process, then updating `application.conf` with the agent endpoint URL.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BridgeAgent** — an AutonomousAgent whose tools are delegated to an external agent framework adapter. Each tool invocation is intercepted by a `before-tool-call` guardrail before execution.
- **RunWorkflow** — orchestrates the per-run lifecycle: accepted → tool-calls in-flight → completed.
- **RunEntity** — an EventSourcedEntity holding the per-run lifecycle and the tool-call audit log.
- **ToolCallGuardrail** — the `before-tool-call` hook; validates each outbound tool request against a per-tool policy before execution.
- **AgentFrameworkAdapter** — a Consumer that proxies tool call results back into the workflow.
- **RunView + RunEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded tool definitions for your own agent framework's real tools.
- `SPEC.md §5` — extend `ToolCallRecord` with framework-specific fields (e.g., `frameworkSessionId`, `toolVersion`, `callerIdentity`).
- `prompts/bridge-agent.md` — narrow the agent's role to a specific task type your framework handles.
- `eval-matrix.yaml` — add a real policy store for the `before-tool-call` guardrail by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a run request → the agent executes tool calls → each tool call appears in the audit log → the run completes with a result.
2. The agent requests a blocked tool — the `before-tool-call` guardrail rejects it — the agent receives a structured rejection and routes around it — the blocked call never reaches the external framework.
3. A run that exceeds its tool-call budget is halted at the guardrail and the entity transitions to `BLOCKED`.
4. Every tool call in the audit log carries the exact request payload sent to the framework, confirming the guardrail cannot retroactively alter the record.

## License

Apache 2.0.
