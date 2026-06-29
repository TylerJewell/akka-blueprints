# Akka Sample: Product Catalog Ad Generation

A single `AdGeneratorAgent` walks a product record through three task phases — **ENRICH → DRAFT → REVIEW** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a focused set of phase-specific tools. The user submits a product catalog entry and receives a finished `ProductAd` ready for placement.

Demonstrates the **sequential-pipeline** coordination pattern with one governance mechanism: a `before-agent-response` guardrail that screens every ad draft for brand-policy violations and prohibited words before the draft is recorded as output.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every enrich, draft, and review tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.product-ad-gen  ~/my-projects/product-ad-gen
cd ~/my-projects/product-ad-gen
```

(Optional) Edit `SPEC.md` to point at a different product catalog seed, a different model provider, or a stricter brand-policy word list.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AdGeneratorAgent** — one AutonomousAgent declaring three Task constants (`ENRICH_PRODUCT`, `DRAFT_AD`, `REVIEW_AD`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **AdGenerationWorkflow** — runs `enrichStep → draftStep → reviewStep → complianceStep`. Each step calls `runSingleTask` and writes the typed result back onto `AdJobEntity` before the next step starts.
- **AdJobEntity** — an EventSourcedEntity holding the per-job lifecycle (`ProductEnriched`, `AdDrafted`, `AdApproved`, `ComplianceScored`).
- **EnrichTools / DraftTools / ReviewTools** — three function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail enforces brand-policy rules on every draft before it is accepted.
- **BrandGuardrail** — the runtime check that backs the ad-compliance contract. A draft containing prohibited language or violating structural brand rules is rejected before the draft is recorded, and the agent is given a structured correction notice.
- **BrandComplianceScorer** — deterministic, rule-based compliance evaluator that runs immediately after `AdApproved` and emits a 1–5 score.
- **AdJobView + AdJobEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded product catalog under `src/main/resources/sample-data/products.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/ad-generator-agent.md` — narrow the agent's role (e.g., constrain it to B2B SaaS products, fashion retail, or industrial parts) by tightening the system prompt and renaming the typed records (`EnrichedProduct`, `AdDraft`, `ProductAd`).
- `SPEC.md §5` — extend the typed outputs (`EnrichedProduct`, `AdDraft`) with industry-specific fields. The brand-compliance guardrail does not need editing for field changes — it checks the draft's prose, not the field names.
- `eval-matrix.yaml` — wire a real brand-similarity evaluator (replace the deterministic stub with an embedding-distance check against approved brand copy) by editing the `G1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a product → `ENRICH` runs → `DRAFT` runs → `REVIEW` runs → a typed `ProductAd` lands in the UI within ~60 s. Every transition is visible in real time.
2. The agent returns a draft containing a prohibited phrase (forced via the mock LLM) → the `before-agent-response` guardrail rejects the draft → the workflow records the rejection event → the agent retries within its iteration budget → the pipeline completes with a compliant ad.
3. Every `ProductAd` emitted has a compliance score visible on the same UI card; ads whose headlines lack a required brand element receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the ENRICH task does not see the draft instructions, and the DRAFT task does not see raw enrichment tool payloads — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
