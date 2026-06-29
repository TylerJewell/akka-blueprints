# Akka Sample: Agents as Tools

A supervisor agent routes incoming tasks to specialist agents exposed as named tools, then assembles the final response. Demonstrates the **delegation-supervisor-workers** coordination pattern where each worker agent is callable as a discrete tool from the supervisor's perspective.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the task stream and the specialist agents are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.agents-as-tools  ~/my-projects/agents-as-tools
cd ~/my-projects/agents-as-tools
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or tool registry.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskSupervisor** — AutonomousAgent that receives an incoming task request, selects the appropriate specialist tool(s) to call, and assembles the final response.
- **WriterAgent** — AutonomousAgent exposed as a tool; handles text drafting tasks.
- **DataAgent** — AutonomousAgent exposed as a tool; handles structured data extraction and summarisation tasks.
- **TaskWorkflow** — Workflow that starts a supervised task session, invokes the supervisor, and persists the result.
- **TaskRecordEntity** — EventSourcedEntity holding each task's lifecycle (queued → processing → completed / failed / rejected).
- **TaskQueue** — EventSourcedEntity that logs each submitted task request for replay and audit.
- **TaskView** — projection the UI streams via SSE.
- **TaskRequestConsumer** — Consumer that starts a TaskWorkflow per submission.
- **TaskSimulator** — TimedAction that drips a sample task request every 60 seconds.
- **QualitySampler** — TimedAction that scores completed tasks on a rubric and emits a quality event.
- **TaskEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample task requests the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — add a third specialist tool (e.g., a `CodeAgent`) and a corresponding Task constant.
- `prompts/supervisor.md` — change the tool-selection heuristic or add tool-call budget limits.
- `eval-matrix.yaml` — no controls are wired by default; add a guardrail if you integrate a real external API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a task request → record enters `QUEUED`, then `PROCESSING`, then `COMPLETED`; UI reflects each transition via SSE.
2. Submit a task whose topic requires both specialist tools → supervisor calls both in sequence; the assembled response references both outputs.
3. Submit a task that cannot be routed → supervisor returns a `REJECTED` outcome with an explanation.

## License

Apache 2.0.
