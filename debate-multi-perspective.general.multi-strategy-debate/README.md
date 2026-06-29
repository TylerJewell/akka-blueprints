# Akka Sample: Multi-Strategy Workflow

Given a query, a coordinator dispatches three independent retrieval and reasoning strategies concurrently — keyword search, semantic vector retrieval, and structured chain-of-thought reasoning — then synthesizes their answers into one authoritative response. Demonstrates the **debate-multi-perspective** coordination pattern with embedded cross-strategy scoring.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — every retrieval strategy is modelled inside the service with Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./debate-multi-perspective.general.multi-strategy-debate  ~/my-projects/multi-strategy-workflow
cd ~/my-projects/multi-strategy-workflow
```

(Optional) Edit `SPEC.md` to change the retrieval strategies, the synthesis rubric, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **StrategyCoordinator** — AutonomousAgent that decomposes a query into three strategy briefs, then synthesizes the strategy results into a final answer.
- **KeywordSearchAgent** — AutonomousAgent that executes keyword-based retrieval and ranks matching passages.
- **SemanticRetrievalAgent** — AutonomousAgent that performs vector-similarity reasoning over the query.
- **ChainOfThoughtAgent** — AutonomousAgent that generates a structured step-by-step reasoning chain and derives an answer.
- **ConsistencyJudge** — AutonomousAgent that scores how much the three strategy answers agree with the synthesized answer.
- **QueryWorkflow** — Workflow that validates the query, fans it out to the three strategy agents in parallel, then calls the coordinator for synthesis.
- **QueryEntity** — EventSourcedEntity holding the query's full lifecycle.
- **QueryView** — projection used by the UI.
- **QueryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample queries the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `StrategyResult` or `SynthesizedAnswer` record fields (e.g., add a `confidenceInterval` field).
- `prompts/chain-of-thought-agent.md` — narrow the reasoning style to a specific domain.
- `eval-matrix.yaml` — add a `before-agent-response` guardrail if you wire a real search index.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → processing enters `RECEIVED`, then `RUNNING`, then `SYNTHESIZED` with three strategy results and one synthesized answer.
2. A query containing blocked content is rejected before any strategy runs; the query enters `REJECTED` and no agent is called.
3. A strategy agent timeout drives the query to `DEGRADED` with the final answer synthesized from whichever strategies returned.
4. The output guardrail blocks a synthesized answer that violates the structural policy; the query enters `BLOCKED`.
5. The eval sampler scores cross-strategy agreement and surfaces the score on the App UI.

## License

Apache 2.0.
