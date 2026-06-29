# Akka Sample: FOMC Event Analyst

A single `FomcAnalystAgent` walks a Federal Reserve policy event through three task phases — **GATHER → INTERPRET → SYNTHESIZE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a phase-specific set of function tools. The user submits an FOMC event identifier and receives a structured `PolicyAnalysis`.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that reviews every synthesized policy analysis for financial output quality before the response is recorded.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every gather / interpret / synthesize tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.fomc-event-analyst  ~/my-projects/fomc-event-analyst
cd ~/my-projects/fomc-event-analyst
```

(Optional) Edit `SPEC.md` to point at a different set of seeded FOMC events, a different model provider, or a richer set of market interpretation tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FomcAnalystAgent** — one AutonomousAgent declaring three Task constants (`GATHER_MARKET_DATA`, `INTERPRET_POLICY_SIGNALS`, `SYNTHESIZE_ANALYSIS`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **FomcPipelineWorkflow** — runs `gatherStep → interpretStep → synthesizeStep → reviewStep`. Each step calls `runSingleTask` and writes the typed result back onto `FomcEventEntity` before the next step starts.
- **FomcEventEntity** — an EventSourcedEntity holding the per-analysis lifecycle (`MarketDataGathered`, `PolicySignalsInterpreted`, `AnalysisSynthesized`, `OutputReviewed`).
- **GatherTools / InterpretTools / SynthesizeTools** — three function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail enforces financial output quality on the final synthesized analysis.
- **FinancialOutputGuardrail** — the runtime check that reviews the agent's synthesized `PolicyAnalysis` before it is recorded. Verifies rate-move attribution, market-impact grounding, and factual completeness; rejects responses that fail the quality bar.
- **PolicyAnalysisView + FomcEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded event set under `src/main/resources/sample-events/events.jsonl` to fit your demo audience or specific FOMC meeting dates.
- `SPEC.md §4` and `prompts/fomc-analyst-agent.md` — narrow the agent's role (e.g., constrain it to rate-decision commentary, to balance-sheet runoff analysis, or to dot-plot interpretation) by tightening the system prompt and renaming the typed records (`MarketSnapshot`, `PolicySignalSet`, `PolicyAnalysis`).
- `SPEC.md §5` — extend the typed outputs with sector-specific fields (e.g., add `yieldCurveImpact` to `PolicyAnalysis`). The guardrail does not need editing — it reviews the final typed output, not field shapes.
- `eval-matrix.yaml` — adjust the `FinancialOutputGuardrail` quality criteria (e.g., require a confidence interval on every rate forecast) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a seeded FOMC event → `GATHER` runs → `INTERPRET` runs → `SYNTHESIZE` runs → a typed `PolicyAnalysis` lands in the UI within ~60 s. Every transition is visible in real time.
2. The agent produces a synthesized analysis that cites a rate-move rationale not grounded in any gathered market data — the `before-agent-response` guardrail rejects it, the workflow retries, and the corrected analysis is recorded.
3. Each task receives only its own typed inputs; the GATHER task does not see synthesis instructions, and the SYNTHESIZE task does not see raw market-data fetches — the workflow's task-chaining is the only path information travels between phases.
4. Every guardrail review outcome (accept or reject with reason) is visible in the UI's review-log strip on the selected analysis card.

## License

Apache 2.0.
