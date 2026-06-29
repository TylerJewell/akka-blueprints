# Akka Sample: Multi-Agent Hello World

A supervisor agent routes an incoming greeting request to two worker agents in parallel — one that generates a greeting message and one that suggests a follow-up action — then merges both outputs into a single response. Demonstrates the **delegation-supervisor-workers** coordination pattern with minimal ceremony.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the greeting pipeline is modelled entirely inside the service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.general.mac-hello-world  ~/my-projects/mac-hello-world
cd ~/my-projects/mac-hello-world
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or response schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GreetingCoordinator** — AutonomousAgent that composes a final response from worker outputs.
- **GreetingWriter** — AutonomousAgent that drafts a greeting message for the given name and context.
- **ActionAdvisor** — AutonomousAgent that suggests a relevant follow-up action.
- **GreetingWorkflow** — Workflow that fans work out to GreetingWriter and ActionAdvisor in parallel, then calls GreetingCoordinator to compose the final reply.
- **GreetingEntity** — EventSourcedEntity holding the greeting request lifecycle.
- **GreetingView** — projection the UI streams via SSE.
- **GreetingEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `GreetingResponse` record fields (e.g., add a `tone` field).
- `prompts/greeting-writer.md` — narrow the worker to a specific language or persona.
- `eval-matrix.yaml` — the controls list is empty for this baseline; add a `before-agent-response` guardrail if you route to a production model.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a name and context → request enters `PENDING`, then `IN_PROGRESS`, then `COMPLETED`.
2. Workers fail-fast → if GreetingWriter times out, the request enters `DEGRADED` with whatever ActionAdvisor returned.
3. The simulator drips a request every 60 seconds so the App UI is non-empty on first load.

## License

Apache 2.0.
