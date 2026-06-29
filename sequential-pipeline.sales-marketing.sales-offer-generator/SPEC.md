# SPEC — sales-offer-generator

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Sales Offer Generator.
**One-line pitch:** The user pastes a customer brief into the App UI; four agents run in sequence — analyze needs, match products, build a pricing strategy, compose an offer — and the UI shows a finished, policy-checked offer document.

## 2. What this blueprint demonstrates

The sequential-pipeline coordination pattern: each agent consumes the prior agent's typed output, with a durable workflow holding the in-flight state between stages. The governance pattern wires a before-agent-response guardrail that checks the composed offer for pricing accuracy and unauthorized discounts before it is stored or shown, and a PII sanitizer that redacts customer identifiers from prompts and logs as the brief flows through the agents.

## 3. User-facing flows

1. The user opens the App UI tab and pastes a customer brief (segment, goals, contact line). On Submit, the service starts a pipeline and returns an `offerId`. The offer appears in `QUEUED`, then `ANALYZED`.
2. The pipeline advances on its own: `MATCHED` once products are selected, `PRICED` once a pricing strategy is built, `COMPOSED` once the offer text is written, then `APPROVED` once the offer passes the pricing-policy guardrail. The UI updates live over SSE.
3. The user clicks an approved offer to see the structured needs, matched products, pricing breakdown, and the rendered offer document.
4. If the composed offer carries an unauthorized discount or a price that does not match the catalog, the offer transitions to `REJECTED` with a reason; the UI shows it.
5. Without any interaction, `RequestSimulator` drips a fresh customer brief every 30 seconds, so offers keep arriving.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| NeedsAnalyst | AutonomousAgent | Brief → structured `CustomerNeeds` | OfferWorkflow | OfferWorkflow |
| ProductMatcher | AutonomousAgent | Needs → `ProductMatch` via a catalog tool | OfferWorkflow | CatalogEndpoint |
| PricingStrategist | AutonomousAgent | Needs + products → `PricingStrategy` | OfferWorkflow | OfferWorkflow |
| OfferComposer | AutonomousAgent | All prior outputs → `OfferDocument` | OfferWorkflow | OfferWorkflow |
| OfferWorkflow | Workflow | Orchestrates analyze → match → price → compose → review | RequestConsumer | OfferEntity, the four agents |
| OfferEntity | EventSourcedEntity | Per-offer lifecycle state | OfferWorkflow | OffersView |
| InboundRequestQueue | EventSourcedEntity | Records each submitted brief | OfferEndpoint, RequestSimulator | RequestConsumer |
| OffersView | View | List-of-offers read model | OfferEntity | OfferEndpoint |
| RequestConsumer | Consumer | Starts a workflow per queued brief | InboundRequestQueue | OfferWorkflow |
| RequestSimulator | TimedAction | Drips a canned brief every 30s | sample-events | InboundRequestQueue |
| OfferEndpoint | HttpEndpoint | `/api` surface — submit, list, single, SSE, metadata | UI | OfferEntity, OffersView |
| CatalogEndpoint | HttpEndpoint | In-process product catalog returning canned SKUs | ProductMatcher | sample-data |
| AppEndpoint | HttpEndpoint | Serves `/` and `/app/*` | browser | static-resources |

Names are used verbatim by `/akka:specify`.

## 5. Data model

See `reference/data-model.md`. Entity state and View row is the `Offer` record; every nullable lifecycle field is `Optional<T>` (Lesson 6). Status enum `OfferStatus`: `QUEUED · ANALYZED · MATCHED · PRICED · COMPOSED · APPROVED · REJECTED`. Events: `NeedsAnalyzed`, `ProductsMatched`, `OfferPriced`, `OfferComposed`, `OfferApproved`, `OfferRejected`. Agent result records: `CustomerNeeds`, `ProductMatch`, `PricingStrategy`, `OfferDocument`.

## 6. API contract

See `reference/api-contract.md` for payload schemas and the SSE event form. Top-level surface:

```
POST /api/offers                   -> { offerId }
GET  /api/offers ?status=...       -> { offers: [Offer, ...] }
GET  /api/offers/{offerId}         -> Offer
GET  /api/offers/sse               -> Server-Sent Events of Offer
GET  /api/catalog ?segment=...     -> canned product list
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (Lesson 17 — no `ui/`, no npm). Browser title: `<title>Akka Sample: Sales Offer Generator</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. The Overview tab is eyebrow ("Overview") + headline (sample type) with no subtitle, then four cards Try it / How it works / Components / API contract. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label colour + edge-label `overflow:visible` CSS overrides (Lesson 24). See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Mechanisms the generated system wires:

- **G1** — a before-agent-response guardrail on `OfferComposer` runs the `PricingPolicy` check over the composed offer: the quoted total must match the priced subtotal-minus-discount, and the discount must not exceed the policy ceiling. A failing offer is blocked and the workflow transitions the offer to `REJECTED` with the policy reason.
- **S1** — a PII sanitizer redacts customer identifiers (email, phone, full name) from the brief before it enters any agent prompt and before any pipeline event is logged. The redaction runs at the system boundary so no downstream prompt or log line carries raw PII.

## 9. Agent prompts

- `prompts/needs-analyst.md` — turns a free-text customer brief into structured needs.
- `prompts/product-matcher.md` — selects catalog products that fit the needs via the catalog tool.
- `prompts/pricing-strategist.md` — builds a competitive pricing strategy within policy bounds.
- `prompts/offer-composer.md` — writes a persuasive offer document from the prior outputs.

## 10. Acceptance

See `reference/user-journeys.md`. Passing journeys:

1. Submit a brief; within ~30s the offer reaches `APPROVED` with a non-empty offer document.
2. A composed offer whose discount exceeds the policy ceiling transitions to `REJECTED` with the policy reason.
3. A brief containing an email and phone number is redacted before it reaches any agent prompt; stored events and logs carry no raw PII.
4. Without interaction, the simulator seeds a brief and a fresh offer pipeline runs to a terminal state.

---

## 11. Implementation directives

The whole SPEC.md — Sections 1–11 — is the input to `/akka:specify @SPEC.md`. This section carries the Akka-specific details Sections 1–10 don't repeat.

```
Create a sample named sales-offer-generator demonstrating the
sequential-pipeline x sales-marketing cell. Runs out of the box (no external
services; the product catalog is modeled in-process). Maven group
io.akka.samples. Maven artifact sales-offer-generator. Java package
io.akka.samples.salesoffergenerator. Akka 3.6.0. HTTP port 9341.

Components to wire (exactly):
- 4 AutonomousAgents:
  - NeedsAnalyst: input a sanitized brief String, returns typed
    CustomerNeeds{segment, painPoints(List<String>), budgetBand,
    desiredOutcomes(List<String>)}.
  - ProductMatcher: input a CustomerNeeds, returns typed ProductMatch{
    products(List<MatchedProduct>)} where MatchedProduct{sku, name, listPrice
    (double), fitReason}. Declares a catalog capability whose tool calls
    GET /api/catalog?segment=...; the tool returns canned SKUs only.
  - PricingStrategist: input CustomerNeeds + ProductMatch, returns typed
    PricingStrategy{subtotal(double), discountPct(double), total(double),
    rationale}.
  - OfferComposer: input CustomerNeeds + ProductMatch + PricingStrategy,
    returns typed OfferDocument{title, body, quotedTotal(double)}. A
    before-agent-response guardrail runs PricingPolicy.check over the result
    before it is returned (quotedTotal equals subtotal*(1-discountPct) within a
    cent; discountPct <= ceiling); a blocked result carries the policy reason.
  Each declares definition() with capability(TaskAcceptance.of(task)
  .maxIterationsPerTask(3)). Never silently downgrade to Agent (Lesson 1).
- 1 Workflow OfferWorkflow with steps analyzeStep -> matchStep -> priceStep ->
  composeStep -> reviewStep. analyzeStep calls
  forAutonomousAgent(NeedsAnalyst.class,...).runSingleTask(...) then
  forTask(taskId).result(...); writes OfferEntity.recordNeeds. matchStep,
  priceStep, composeStep follow the same agent-call shape, writing recordMatch,
  recordPricing, recordComposition. reviewStep runs the in-process
  PricingPolicy over the composed offer; on pass calls OfferEntity.approve, on
  fail OfferEntity.reject(reason). Override settings() with stepTimeout(60s) on
  analyzeStep, matchStep, priceStep, composeStep (Lesson 4); WorkflowSettings is
  nested in Workflow, no import (Lesson 5).
- 1 EventSourcedEntity OfferEntity holding an Offer record: id, brief
  (Optional<String>), OfferStatus enum, and Optional lifecycle fields
  analyzedAt, needs, matchedAt, products, pricedAt, pricing, composedAt,
  offerTitle, offerBody, quotedTotal, approvedAt, rejectedAt, rejectReason.
  Events: NeedsAnalyzed, ProductsMatched, OfferPriced, OfferComposed,
  OfferApproved, OfferRejected. Commands: recordNeeds, recordMatch,
  recordPricing, recordComposition, approve, reject, getOffer. emptyState()
  returns Offer.initial("") with NO commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue, single instance "default", one
  command enqueueBrief(brief) emitting BriefQueued.
- 1 View OffersView with row type Offer, table updater consuming OfferEntity
  events. ONE query getAllOffers: SELECT * AS offers FROM offers_view. No WHERE
  status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side.
- 1 Consumer RequestConsumer subscribed to InboundRequestQueue events; on each
  BriefQueued runs the PiiSanitizer over the brief then starts an OfferWorkflow
  with a fresh UUID and the sanitized brief.
- 1 TimedAction RequestSimulator (every 30s) reads the next line from
  src/main/resources/sample-events/briefs.jsonl and calls
  InboundRequestQueue.enqueueBrief.
- 3 HttpEndpoints: OfferEndpoint at /api (offers submit, offers list filtered
  client-side from getAllOffers, single offer, SSE stream, three
  /api/metadata/* endpoints serving files from src/main/resources/metadata/).
  CatalogEndpoint at /api/catalog returning canned products from
  src/main/resources/sample-data/ filtered by segment. AppEndpoint serving
  / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- OfferTasks.java declaring four Task<R> constants: ANALYZE (resultConformsTo
  CustomerNeeds), MATCH (ProductMatch), PRICE (PricingStrategy), COMPOSE
  (OfferDocument). Required for every AutonomousAgent (Lesson 7).
- CustomerNeeds, ProductMatch, MatchedProduct, PricingStrategy, OfferDocument
  records (see reference/data-model.md).
- PricingPolicy.java — pure helper: returns pass/fail + reason for a composed
  offer (quotedTotal matches subtotal*(1-discountPct) within a cent; discountPct
  <= MAX_DISCOUNT ceiling). Used both as the before-agent-response guardrail body
  and the reviewStep gate.
- PiiSanitizer.java — pure helper: redacts email, phone, and full-name patterns
  from a brief String, returning the redacted text. Applied in RequestConsumer
  before the brief enters any prompt or event.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9341 and agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-events/briefs.jsonl with 8 canned customer-brief
  lines (each carrying a synthetic email/phone so the PII sanitizer is exercised).
- src/main/resources/sample-data/catalog.json with 6-10 canned products the
  CatalogEndpoint serves, segment-tagged.
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md} —
  classpath copies for the metadata endpoints.
- eval-matrix.yaml + risk-survey.yaml at the project root.
- README.md at the project root (Lesson 20 — no governance-mechanisms section,
  no configuration section).
- src/main/resources/static-resources/index.html — single self-contained file,
  inline CSS + JS, runtime CDN imports for markdown + YAML allowed. Five tabs.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, detect which of ANTHROPIC_API_KEY / OPENAI_API_KEY /
  GOOGLE_AI_GEMINI_API_KEY is set. If exactly one, default application.conf's
  model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (NeedsAnalyst -> CustomerNeeds, ProductMatcher -> ProductMatch,
  PricingStrategist -> PricingStrategy, OfferComposer -> OfferDocument; see Mock
  LLM shapes below). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the reference.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option (a)), written to
src/main/resources/mock-responses/{needs-analyst,product-matcher,
pricing-strategist,offer-composer}.json, 4-6 entries each:
- needs-analyst.json: CustomerNeeds objects {segment, painPoints[], budgetBand,
  desiredOutcomes[]}.
- product-matcher.json: ProductMatch objects {products:[{sku,name,listPrice,
  fitReason}]} referencing catalog SKUs.
- pricing-strategist.json: PricingStrategy objects {subtotal, discountPct, total,
  rationale} where total equals subtotal*(1-discountPct) and discountPct is
  within the policy ceiling.
- offer-composer.json: OfferDocument objects {title, body, quotedTotal} where
  quotedTotal matches the paired PricingStrategy total. Include at least one
  entry whose quotedTotal violates the policy so journey 2 (REJECTED) is
  reproducible.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: explicit stepTimeout on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable lifecycle field on the Offer row record.
- Lesson 7: every AutonomousAgent has its OfferTasks Task<R> constant.
- Lesson 8: verify model names are current before locking.
- Lesson 9: run command is "/akka:build" (Claude Code slash command), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9341 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: mermaid state-label colour + edge-label overflow:visible CSS
  overrides and theme variables in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — a missing API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
