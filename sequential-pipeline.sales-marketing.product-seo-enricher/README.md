# Akka Sample: Product SEO Enricher

A single `SeoAgent` walks a product record through three task phases — **FETCH → ANALYZE → ENRICH** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a product identifier and receives an enriched `ProductEnrichment` containing keyword candidates, competitor signals, and a recommended meta description.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that blocks browser-tool calls when the phase's preconditions are not yet satisfied.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. All FETCH / ANALYZE / ENRICH tools are implemented in-process. Search-result pages are served from a bundled fixture corpus — no live internet calls required.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.product-seo-enricher  ~/my-projects/product-seo-enricher
cd ~/my-projects/product-seo-enricher
```

(Optional) Edit `SPEC.md` to swap the seeded product catalog under `src/main/resources/sample-data/products.jsonl`, point at a different model provider, or extend the keyword-extraction heuristics inside `AnalyzeTools`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SeoAgent** — one AutonomousAgent declaring three Task constants (`FETCH_SERP`, `ANALYZE_SERP`, `WRITE_ENRICHMENT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **EnrichmentPipelineWorkflow** — runs `fetchStep → analyzeStep → enrichStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `EnrichmentEntity` before the next step starts.
- **EnrichmentEntity** — an EventSourcedEntity holding the per-product lifecycle (`SerpFetched`, `SerpAnalyzed`, `EnrichmentWritten`, `QualityScored`).
- **FetchTools / AnalyzeTools / EnrichTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **BrowserGuardrail** — the runtime check that enforces browser-tool access. A tool call that accesses search-result data before the FETCH phase has recorded its `SerpResult` is rejected before the tool runs.
- **QualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `EnrichmentWritten` and emits a 1–5 quality score.
- **EnrichmentView + EnrichmentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded product set under `src/main/resources/sample-data/products.jsonl` to fit your catalog.
- `SPEC.md §4` and `prompts/seo-agent.md` — narrow the agent's role (e.g., restrict to a single product category, add locale awareness) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend `KeywordCandidate` and `CompetitorSignal` with vertical-specific fields. The phase-gating guardrail checks phase preconditions, not record field shapes.
- `eval-matrix.yaml` — replace the deterministic keyword-coverage scorer with a real search-volume lookup by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a product → `FETCH` runs → `ANALYZE` runs → `ENRICH` runs → a typed `ProductEnrichment` lands in the UI within ~60 s. Every transition is visible in real time.
2. A tool from a later phase is called out of order (forced via the mock LLM) → `BrowserGuardrail` rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. Every `ProductEnrichment` emitted has a quality score visible on the same UI card; enrichments that produce no keyword candidates receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the FETCH task does not see enrichment instructions, and the ENRICH task does not see raw SERP page bytes — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
