# Akka Sample: AI Ads Generator

A single `AdCreatorAgent` walks a product URL through three task phases — **SCRAPE → COPY → VISUAL** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a dedicated set of phase-specific tools. The user submits a product page URL and receives a structured `AdPackage` containing finalized ad copy and an image-generation prompt.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-agent-response` guardrail that screens outbound ad copy for brand-safety violations before it reaches the response, and a `before-tool-call` guardrail that enforces a scraping policy before any browser-automation tool fires.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every scrape, copy, and visual tool is implemented in-process using static product-data fixtures — no external browser or image-generation service is required to run the sample.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.ad-creator-pipeline  ~/my-projects/ad-creator-pipeline
cd ~/my-projects/ad-creator-pipeline
```

(Optional) Edit `SPEC.md` to point at a different product seed list, a different model provider, or a richer set of copy-style guidelines in `prompts/ad-creator-agent.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AdCreatorAgent** — one AutonomousAgent declaring three Task constants (`SCRAPE_PRODUCT`, `DRAFT_COPY`, `GENERATE_VISUAL`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **AdPipelineWorkflow** — runs `scrapeStep → copyStep → visualStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `AdJobEntity` before the next step starts.
- **AdJobEntity** — an EventSourcedEntity holding the per-job lifecycle (`ProductScraped`, `CopyDrafted`, `VisualGenerated`, `AdEvaluated`).
- **ScrapeTools / CopyTools / VisualTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase and that browser-automation tools satisfy the scraping policy.
- **ScrapingPolicyGuardrail** — the runtime check that backs the scraping-policy contract. A browser tool whose target URL does not satisfy the allow-list (carried on the `AdJobEntity`) is rejected before the tool runs.
- **BrandSafetyGuardrail** — the `before-agent-response` hook. Screens the agent's outbound ad copy for disallowed claims, competitor mentions, and prohibited product categories before the response is accepted.
- **AdQualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `VisualGenerated` and emits a 1–5 quality score.
- **AdJobView + AdJobEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded product list under `src/main/resources/sample-events/products.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/ad-creator-agent.md` — narrow the agent's role (e.g., constrain it to a single product category, a specific brand voice, or a regulatory-approved disclaimer set) by tightening the system prompt and renaming the typed records (`ProductProfile`, `AdCopy`, `VisualSpec`).
- `SPEC.md §5` — extend the typed outputs with industry-specific fields (e.g., add a `regulatoryDisclaimer` field to `AdCopy` for financial-services or pharmaceutical advertising).
- `eval-matrix.yaml` — wire a real brand-safety classifier (replace the deterministic stub with a model call against your brand guidelines) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a product URL → `SCRAPE` runs → `COPY` runs → `VISUAL` runs → a typed `AdPackage` lands in the UI within ~60 s. Every transition is visible in real time.
2. A scraping tool targets a URL outside the allow-list (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries with a permitted URL → the pipeline completes correctly.
3. The agent's COPY-phase output contains a disallowed competitor mention → the `before-agent-response` guardrail blocks the response → the agent rewrites the copy without the violation → the pipeline continues.
4. Every `AdPackage` emitted has an on-decision quality score visible on the same UI card; packages whose copy cites no product attribute from the scraped profile receive a score ≤ 2 and are flagged.

## License

Apache 2.0.
