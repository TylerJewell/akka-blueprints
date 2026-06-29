# Akka Sample: Multi-LLM Load-Balanced Agent

A single CodeAgent routes every prompt through a `LiteLLMRouterModel` that distributes calls across OpenAI `gpt-4o-mini` and Anthropic Claude using simple-shuffle ordering. The system records each call's latency, provider, and response quality so operators can monitor comparative performance across providers over time.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a periodic performance evaluator that aggregates per-call quality signals and surfaces provider-level trends without blocking any individual call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have keys, supply them via one of: existing shell env vars (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software required. The router and the call-log store run in-process.

## Generate the system

```sh
cp -r ./single-agent.general.multi-provider-router  ~/my-projects/multi-provider-router
cd ~/my-projects/multi-provider-router
```

(Optional) Edit `SPEC.md` to add a third provider, change the shuffle strategy to weighted-round-robin, or extend `CallRecord` with additional quality dimensions.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RouterAgent** — an AutonomousAgent that receives a prompt and dispatches it to one of the configured providers via the `LiteLLMRouterModel` shuffle.
- **RoutingWorkflow** — orchestrates dispatch → record → optional eval-trigger per prompt.
- **CallEntity** — an EventSourcedEntity holding the per-call lifecycle and its performance record.
- **PerformanceMonitor** — a Consumer that subscribes to `CallCompleted` events and feeds rolling quality signals into the monitor view.
- **CallView + RouterEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the two seeded providers for a different pair, or add a third.
- `SPEC.md §5` — extend `CallRecord` with domain-specific quality dimensions (e.g. `groundedness`, `citation_count`, `refusal_rate`).
- `prompts/router-agent.md` — narrow the agent's task (a summarization deployer would constrain the prompt template; a code-gen deployer would add output-format instructions).
- `eval-matrix.yaml` — wire a real quality scorer (e.g. an embedding cosine-similarity check) by naming it under the performance-monitor mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → the router selects a provider → the call completes → the result and latency appear in the UI.
2. Multiple calls in sequence distribute across both providers — neither provider handles all traffic.
3. The performance monitor accumulates call data; the Eval Matrix tab shows rolling quality scores per provider.
4. A simulated high-latency call is recorded correctly; the monitor flags the provider's P95 latency above threshold in the next evaluation window.

## License

Apache 2.0.
