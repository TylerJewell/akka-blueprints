# Akka Sample: Hot-Reload Agent Persona via Configuration Store

A durable agent subscribes to a Configuration Store (backed by Redis in the integration tier) and watches three keys — `agent_role`, `agent_goal`, and `agent_instructions`. When any key changes, the agent's persona updates in-place, with no restart required. An optional `agent_model` key triggers a live model swap, also without restart.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a CI-side change-control gate that validates persona payloads before they reach the store, a behavioral revalidation evaluator that fires on every live persona swap, and a human-on-the-loop monitoring hook that surfaces post-swap observations to a designated operator channel.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- **Integration tier:** A Redis-compatible instance reachable by the service. A local `docker run -p 6379:6379 redis:7-alpine` works for development. The Akka Configuration Store client reads `REDIS_URL` from the environment; set it before starting the service. A blank `REDIS_URL` causes the service to fall back to an in-process stub that simulates watch callbacks for the seeded persona fixtures.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.akka-persona-hot-reload  ~/my-projects/persona-hot-reload
cd ~/my-projects/persona-hot-reload
```

(Optional) Edit `SPEC.md` to point at a different set of seeded persona fixtures (Section 3), or swap the behavioral-revalidation probe set in `eval-matrix.yaml`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PersonaAgent** — an AutonomousAgent whose definition is rebuilt on every persona change; the running instance answers queries against the current persona without restart.
- **PersonaWatchWorkflow** — a long-running Workflow that subscribes to the Configuration Store, receives change notifications, validates the incoming payload, updates the agent definition, and triggers revalidation.
- **PersonaEntity** — an EventSourcedEntity holding the persona lifecycle: pending → active, with a change log.
- **PersonaChangeConsumer** — a Consumer that reacts to `PersonaActivated` events and kicks off the post-swap monitoring window.
- **PersonaView + PersonaEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded persona fixtures for your own deployment context (e.g., a claims-handling assistant or a customer-support agent). Each fixture is a named set of `agent_role`, `agent_goal`, and `agent_instructions` values.
- `SPEC.md §5` — extend `PersonaSnapshot` with domain-specific metadata (e.g., `complianceOwner`, `approvedModelVersions`).
- `prompts/persona-agent.md` — narrow or widen the agent's allowed instruction topics.
- `eval-matrix.yaml` — replace the seeded behavioral probes with production-relevant questions for your deployment.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A persona change pushed to the store applies in-place; a query sent immediately after returns an answer from the new persona without any service restart.
2. A configuration payload missing required fields is rejected by the gate before it reaches the agent; the persona stays on the previous version.
3. A persona swap triggers the behavioral revalidation suite; probe failures are surfaced in the UI and on the monitoring hook.
4. The raw `agent_instructions` value never appears in the agent's system-prompt call log in its pre-validation, untruncated form; only the validated, active snapshot does.

## License

Apache 2.0.
