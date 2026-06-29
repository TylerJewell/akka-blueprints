# Architecture — product-catalog-ad-generation

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `AdJobEndpoint` accepts a `{productName}` POST, writes `AdJobCreated` onto `AdJobEntity`, and starts `AdGenerationWorkflow` keyed by `"pipeline-" + jobId`. The workflow's first step (`enrichStep`) emits `EnrichStarted`, then calls `AdGeneratorAgent` with `TaskDef.taskType(ENRICH_PRODUCT)` and a `phase = ENRICH` metadata tag. The agent invokes `EnrichTools.lookupCategory` and `EnrichTools.inferAttributes`; these responses pass through `BrandGuardrail` — though brand rules apply only to DRAFT and REVIEW responses, the guardrail is registered on all agent responses and routes accordingly. Once the agent returns an `EnrichedProduct`, the workflow writes `ProductEnriched` onto the entity and advances to `draftStep` — same pattern, the DRAFT task carries `phase = DRAFT`. Then `reviewStep` runs with `phase = REVIEW`. After `AdApproved` lands, `complianceStep` runs `BrandComplianceScorer` over the recorded `(ProductAd, AdDraft, EnrichedProduct)` triple — no LLM call — and writes `ComplianceScored`. `AdJobView` projects every event into a read-model row; `AdJobEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `BrandComplianceScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `enrichStep` and `draftStep`, the workflow writes `ProductEnriched` onto the entity. The next step then reads `enrichedProduct` from the entity to build the DRAFT task's instruction context. The agent never sees enrich-phase context inside the draft task's conversation; the typed handoff is the only path information travels.
2. Every DRAFT and REVIEW response passes through `BrandGuardrail`. The guardrail inspects the full structured response — headline, copy fields, CTA, and placement sizes — as a unit. A violation on any field produces a structured `BrandRejection` listing every failing field, rule, and found value. The agent gets exactly the information it needs to self-correct.

The agent calls are bounded by per-step timeouts (60 s on enrich / draft / review). `complianceStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → ENRICHING → ENRICHED → DRAFTING → DRAFTED → REVIEWING → APPROVED → SCORED`.
- Three failure transitions land in `FAILED`: an agent error during `ENRICHING`, `DRAFTING`, or `REVIEWING`. A `FAILED` job's prior data is preserved on the entity — the UI shows the partial state.
- `GuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `PUBLISHED` or `PLACED` state. The ad is advisory output; a human marketing reviewer acts outside the system. The blueprint deliberately stops at `SCORED`.

## Entity model

`AdJobEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the compliance result, the guardrail audit, the failure, and the initial creation. `AdJobView` projects every event into a row used by the UI. `AdGenerationWorkflow` both reads (`getAdJob`) and writes (`startEnrich`, `recordEnrichment`, `startDraft`, `recordDraft`, `startReview`, `recordAd`, `recordCompliance`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `AdGeneratorAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Governance flow

For any ad job that lands in the entity log, the product passed through:

1. **Brand-policy guardrail** — every DRAFT and REVIEW agent response is filtered. A response containing prohibited words or missing a required headline element is rejected before `AdDrafted` or `AdApproved` is written; a `GuardrailRejected` event records the violation for audit.
2. **AdGeneratorAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Compliance evaluator** — every emitted `ProductAd` gets a 1–5 compliance score. Headline element presence, prohibited-word absence, CTA slug validity, and placement parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not compute a numeric score; the scorer does not block drafts. Removing one of them opens an explicit gap the other does not silently cover.
