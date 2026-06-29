# Akka Sample: OpenAI Agents-as-Tools Composition

A top-level supervisor agent exposes a set of specialist sub-agents as callable tools. The supervisor decides which sub-agent to invoke, the Workflow guarantees durable execution, and a before-tool-call guardrail gates every sub-agent invocation before it is dispatched.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the sub-agent tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.akka-agents-as-tools  ~/my-projects/agents-as-tools
cd ~/my-projects/agents-as-tools
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or task routing logic.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **Supervisor** — AutonomousAgent that receives a task request, selects which sub-agent tool to invoke, and assembles a final response.
- **SummarizerAgent** — AutonomousAgent exposed as a tool that produces concise text summaries.
- **ClassifierAgent** — AutonomousAgent exposed as a tool that assigns category labels to text.
- **TranslatorAgent** — AutonomousAgent exposed as a tool that translates text between languages.
- **CompositionWorkflow** — Workflow that drives the supervisor's tool-call loop with durable step tracking.
- **TaskRequestEntity** — EventSourcedEntity holding the task request's full lifecycle.
- **TaskRequestQueue** — EventSourcedEntity that logs each inbound request for replay and audit.
- **CompositionView** — projection the UI streams via SSE.
- **TaskRequestConsumer** — Consumer that starts a workflow per submitted request.
- **RequestSimulator** — TimedAction that drips sample tasks every 60 seconds.
- **EvalSampler** — TimedAction that scores completed tasks every 5 minutes.
- **CompositionEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample tasks the simulator drips, or remove it entirely.
- `SPEC.md §5` — adjust the `TaskResult` record fields (e.g., add `confidenceScore`).
- `prompts/supervisor.md` — change the routing policy for which sub-agent gets called first.
- `eval-matrix.yaml` — tighten the before-tool-call guardrail to inspect the tool arguments.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task request → request enters `QUEUED`, then `IN_PROGRESS`, then `COMPLETED`.
2. Guardrail blocks a prohibited tool call → workflow routes to `BLOCKED` before the sub-agent runs.
3. Eval-event sampling captures one completed task and surfaces the score on the App UI.

## License

Apache 2.0.
