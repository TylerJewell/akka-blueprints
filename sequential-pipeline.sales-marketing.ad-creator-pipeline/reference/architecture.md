# Architecture — ad-creator-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `AdJobEndpoint` accepts a `{productUrl}` POST, writes `AdJobCreated` onto `AdJobEntity`, and starts `AdPipelineWorkflow` keyed by `"pipeline-" + adJobId`. The workflow's first step (`scrapeStep`) emits `ScrapeStarted`, then calls `AdCreatorAgent` with `TaskDef.taskType(SCRAPE_PRODUCT)` and a `phase = SCRAPE` metadata tag. The agent invokes `ScrapeTools.fetchProductPage` and `ScrapeTools.extractAttributes`; every tool call passes through `ScrapingPolicyGuardrail` first, which checks both the target URL against the allow-list and the phase-status precondition. Once the agent returns a `ProductProfile`, the workflow writes `ProductScraped` onto the entity and advances to `copyStep` — same pattern, the COPY task carries `phase = COPY`. Every response the agent emits during the COPY task passes through `BrandSafetyGuardrail` before the workflow accepts it. Then `visualStep` runs with `phase = VISUAL`. After `VisualGenerated` lands, `evalStep` assembles an `AdPackage` from the three phase results, runs `AdQualityScorer` — no LLM call — and writes `AdEvaluated`. `AdJobView` projects every event into a read-model row; `AdJobEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `AdQualityScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `scrapeStep` and `copyStep`, the workflow writes `ProductScraped` onto the entity. The next step then reads `profile` from the entity to build the COPY task's instruction context. The agent never sees scrape-phase context inside the copy task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `ScrapingPolicyGuardrail`. The guardrail reads the in-flight task's `phase` metadata, the current `AdJobEntity.status`, and for browser tools the target URL. A misordered call or a disallowed URL is rejected before the tool body executes.
3. Every agent response during the COPY task is filtered through `BrandSafetyGuardrail`. The guardrail is stateless — it reads only the response text and the brand policy file. A response containing a violation is blocked; the agent must rewrite within its iteration budget.

The agent calls themselves are bounded by per-step timeouts (60 s on scrape / copy / visual). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → SCRAPING → SCRAPED → COPYING → COPIED → GENERATING_VISUAL → VISUAL_GENERATED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `SCRAPING`, `COPYING`, or `GENERATING_VISUAL`. A `FAILED` job's prior data is preserved on the entity — the UI shows the partial state.
- `ScrapingRejected` and `BrandSafetyRejected` are side-events recorded for audit; they do not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` state. The ad package is a draft for human review; the marketer consumes it and decides whether to publish outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`AdJobEntity` is the source of truth. It emits eleven event types — three lifecycle starts, three lifecycle completions, the evaluation, two audit-only rejection events, the initial creation, and the failure. `AdJobView` projects every event into a row used by the UI. `AdPipelineWorkflow` both reads (`getAdJob`) and writes (`startScrape`, `recordProfile`, `startCopy`, `recordCopy`, `startVisual`, `recordVisual`, `recordEvaluation`, `recordScrapingRejection`, `recordBrandSafetyRejection`, `fail`) on the entity. The relationship between `AdCreatorAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any ad package that lands in the entity log, the product URL passed through:

1. **Scraping-policy guardrail** — every browser-automation tool call is filtered. A `fetchProductPage` call targeting a URL outside the allow-list is rejected before the network request is issued; a `ScrapingRejected` event records the violation for audit. Phase-order is enforced by the same hook.
2. **AdCreatorAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Brand-safety guardrail** — every agent response during the COPY task is screened. A response containing a disallowed claim is blocked; a `BrandSafetyRejected` event records the violation for audit.
4. **On-decision evaluator** — every assembled `AdPackage` gets a 1–5 quality score. Attribute grounding, call-to-action presence, visual-copy alignment, and variant coverage are each worth one point on a base of 1.

Each step is independent. The scraping-policy guardrail does not check brand copy; the brand-safety guardrail does not check URL policy; the evaluator checks neither. Removing any one of them opens an explicit gap the others do not silently cover.
