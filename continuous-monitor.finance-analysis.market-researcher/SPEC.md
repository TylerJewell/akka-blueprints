# SPEC — market-researcher

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Sector Market Researcher.
**One-line pitch:** A background worker ingests research items, classifies their credit and risk relevance, synthesizes structured flags, and validates every flag output before it reaches a downstream desk.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`RelevanceClassifierAgent` and `SynthesisAgent`). Specifically:

- A **before-agent-response guardrail** on `SynthesisAgent` intercepts the outgoing `ResearchFlag` before it is committed and delivered. If the flag is missing a `sourceReference` or has no `confidenceLevel`, the guardrail rejects the response and routes the item to `REVIEW_REQUIRED` rather than delivering it downstream.
- An **eval-periodic** sampler runs every 24 hours, replaying a sample of delivered flags through the current `RelevanceClassifierAgent` and scoring flag precision/recall against a ground-truth label file.

Together these ensure: AI-generated flags have minimum required structure before distribution, and the system's classification quality is measured continuously.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live research feed: every ingested item, its classification, and (when synthesised) the structured flag.
2. `ResearchPoller` (TimedAction) ticks every 20 s and inserts new simulated items into `ResearchItemQueue`. (A simulator style — drips canned items representing news headlines, 8-K filing abstracts, and broker notes.)
3. For each new item: `ResearchWorkflow` invokes `RelevanceClassifierAgent`. If the classification is `NOISE`, the item terminates as `ARCHIVED` immediately — no synthesis call.
4. For `CREDIT_RELEVANT`, `RISK_RELEVANT`, or `MARKET_COLOR` items, `SynthesisAgent` produces a `ResearchFlag`.
5. The before-agent-response guardrail validates the flag. If it passes, the item transitions to `DELIVERED`. If the flag is incomplete, the item transitions to `REVIEW_REQUIRED` and appears in a banner in the UI.
6. `EvalRunner` (TimedAction) ticks every 24 hours, samples `DELIVERED` items without `evalScore`, calls `FlagEvalJudge`, and writes a precision/recall score back via `EvalScored`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchPoller` | `TimedAction` | Drips simulated research items every 20 s. | scheduler | `ResearchItemQueue` |
| `ResearchItemQueue` | `EventSourcedEntity` | Append-only audit log of `ItemReceived` events. | `ResearchPoller`, `ResearchEndpoint` | `ResearchWorkflow` |
| `RelevanceClassifierAgent` | `Agent` (typed, not autonomous) | Classifies each item as `CREDIT_RELEVANT`, `RISK_RELEVANT`, `MARKET_COLOR`, or `NOISE`. | invoked by Workflow | returns ClassificationResult |
| `SynthesisAgent` | `AutonomousAgent` | Synthesizes relevant items into a structured `ResearchFlag`. | invoked by Workflow | returns ResearchFlag |
| `ResearchWorkflow` | `Workflow` | Per-item orchestration: classify → (if relevant) synthesize → guardrail check → deliver or review. | `ResearchItemQueue` events | `ResearchItemEntity` |
| `ResearchItemEntity` | `EventSourcedEntity` | Full per-item lifecycle: ingested → classified → synthesised → delivered / review-required / archived. | `ResearchWorkflow` | `ResearchView` |
| `ResearchView` | `View` | Read-model row per item. | `ResearchItemEntity` events | `ResearchEndpoint` |
| `EvalRunner` | `TimedAction` | Every 24 h, samples delivered items without scores; calls judge; writes `EvalScored`. | scheduler | `ResearchItemEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/research/*` — list, get, acknowledge-review, SSE. | — | `ResearchView`, `ResearchItemEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record ResearchItem(String itemId, String source, String headline,
                    String abstract_, Instant publishedAt) {}

record ClassificationResult(RelevanceCategory category, String confidence,
                             String reason) {}
enum RelevanceCategory { CREDIT_RELEVANT, RISK_RELEVANT, MARKET_COLOR, NOISE }

record ResearchFlag(String flagType, String sourceReference, String confidenceLevel,
                    String narrative, Instant generatedAt) {}

record ResearchItemState(
    String itemId,
    ResearchItem raw,
    Optional<ClassificationResult> classification,
    Optional<ResearchFlag> flag,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    ItemStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ItemStatus {
    INGESTED, CLASSIFIED, SYNTHESISED, DELIVERED, REVIEW_REQUIRED, ARCHIVED
}
```

Events on `ResearchItemEntity`: `ItemReceived`, `ItemClassified`, `ItemSynthesised`, `ItemDelivered`, `ItemReviewRequired`, `ItemArchived`, `EvalScored`.
Events on `ResearchItemQueue`: `ItemReceived` (the audit log entry).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/research` — list all items. Optional `?status=…`.
- `GET /api/research/{id}` — one item.
- `POST /api/research/{id}/acknowledge` — body `{ reviewedBy, notes? }` → acknowledges a `REVIEW_REQUIRED` item; transitions to `ARCHIVED`.
- `GET /api/research/sse` — Server-Sent Events for item changes.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Sector Market Researcher</title>`.

App UI tab is the most distinctive: it shows a **two-panel layout** — left panel is the live research feed with classification chips and status badges; right panel shows the selected item's flag details and eval score. A `REVIEW_REQUIRED` banner appears at the top of the list when any item is blocked by the guardrail.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** on `SynthesisAgent`: intercepts the `ResearchFlag` after the agent returns it and before it is committed. If `sourceReference` is empty or `confidenceLevel` is absent, the guardrail blocks the response and emits `ItemReviewRequired`. Analyst acknowledges via `POST /api/research/{id}/acknowledge`.
- **E1 — eval-periodic** (every 24 hours): samples `DELIVERED` items without `evalScore`. Calls `FlagEvalJudge` with the item's `ClassificationResult` and `ResearchFlag` alongside a ground-truth label. Writes `EvalScored` with score 1–5 and precision/recall estimates.

## 9. Agent prompts

- `RelevanceClassifierAgent` → `prompts/relevance-classifier.md`. Typed classifier. Always returns one of the four `RelevanceCategory` values.
- `SynthesisAgent` → `prompts/synthesis.md`. Produces a `ResearchFlag` — must always populate `sourceReference` and `confidenceLevel`.
- `FlagEvalJudge` (used by `EvalRunner`) → `prompts/flag-eval-judge.md`. Compares the flag against a ground-truth label; outputs score 1–5 plus precision/recall estimates.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an item; it appears in the UI within 20 s; transitions INGESTED → CLASSIFIED → SYNTHESISED → DELIVERED.
2. **J2** — A `NOISE` item terminates as ARCHIVED without any synthesis call.
3. **J3** — A flag missing `sourceReference` is blocked by the guardrail; item transitions to REVIEW_REQUIRED; analyst acknowledges; item transitions to ARCHIVED.
4. **J4** — EvalRunner scores at least one DELIVERED item within the first 24-hour cycle; the score and rationale appear in the UI.
5. **J5** — Debug log confirms the LLM synthesis call receives the normalised abstract, not the raw item (Lesson 4 — no raw PII surface).

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named market-researcher demonstrating the continuous-monitor ×
finance-analysis cell. Runs out of the box (in-memory research feed; no real
data-vendor integration). Maven group io.akka.samples. Artifact
continuous-monitor-finance-analysis-market-researcher. Java package
io.akka.samples.sectormarketresearcher. HTTP port 9936.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RelevanceClassifierAgent — system prompt loaded
  from prompts/relevance-classifier.md. Input: ResearchItem{itemId, source,
  headline, abstract_, publishedAt}. Output: ClassificationResult{category:
  RelevanceCategory (enum CREDIT_RELEVANT/RISK_RELEVANT/MARKET_COLOR/NOISE),
  confidence: "high"|"medium"|"low", reason: String}. Defaults to MARKET_COLOR
  under uncertainty rather than NOISE.

- 1 AutonomousAgent SynthesisAgent — definition() with capability(
  TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt from
  prompts/synthesis.md. Input: ResearchItem + ClassificationResult. Output:
  ResearchFlag{flagType: String ("CREDIT"|"RISK"|"COLOR"), sourceReference: String,
  confidenceLevel: String ("HIGH"|"MEDIUM"|"LOW"), narrative: String,
  generatedAt: Instant}. MUST populate sourceReference and confidenceLevel; the
  guardrail rejects flags where either is blank or null.

- 1 AutonomousAgent FlagEvalJudge — definition() with capability(
  TaskAcceptance.of(EVAL_FLAG).maxIterationsPerTask(2)). System prompt from
  prompts/flag-eval-judge.md. Input: ResearchItem + ClassificationResult +
  ResearchFlag + groundTruthLabel: String. Output: FlagEvalResult{score: Integer
  1–5, precisionEstimate: double, recallEstimate: double, rationale: String}.

- 1 Workflow ResearchWorkflow per itemId with steps:
  classifyStep -> branchStep -> synthesiseStep -> guardrailStep -> deliverStep.
  classifyStep wraps RelevanceClassifierAgent with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(10)). synthesiseStep wraps SynthesisAgent with
  stepTimeout 30s. branchStep: if category == NOISE, emit ItemArchived and end.
  guardrailStep: inspect ResearchFlag; if sourceReference is blank OR
  confidenceLevel is blank, emit ItemReviewRequired and end; otherwise continue
  to deliverStep. deliverStep emits ItemDelivered.

- 2 EventSourcedEntities:
  * ResearchItemQueue — append-only audit log. Command receive(ResearchItem) emits
    ItemReceived{raw}. emptyState() returns initial state without commandContext().
  * ResearchItemEntity (one per itemId) — full per-item lifecycle. State
    ResearchItemState{itemId, raw: ResearchItem, Optional<ClassificationResult>
    classification, Optional<ResearchFlag> flag, Optional<Integer> evalScore,
    Optional<String> evalRationale, ItemStatus status, Instant createdAt,
    Optional<Instant> finishedAt}.
    ItemStatus enum: INGESTED, CLASSIFIED, SYNTHESISED, DELIVERED,
    REVIEW_REQUIRED, ARCHIVED.
    Events: ItemReceived, ItemClassified, ItemSynthesised, ItemDelivered,
    ItemReviewRequired, ItemArchived, EvalScored.
    Commands: registerIncoming, attachClassification, attachFlag, markDelivered,
    markReviewRequired, markArchived, recordEval, getItem.
    emptyState() without commandContext() reference.

- 1 View ResearchView with row type ResearchItemRow (mirrors ResearchItemState).
  Table updater consumes ResearchItemEntity events. ONE query getAllItems
  SELECT * AS items FROM research_item_view. No WHERE status filter — caller
  filters client-side.

- 2 TimedActions:
  * ResearchPoller — every 20s, reads next line from src/main/resources/
    sample-events/research-items.jsonl and calls ResearchItemQueue.receive.
  * EvalRunner — every 24 hours (configurable via EVAL_RUNNER_SECONDS env var
    for testing), queries ResearchView.getAllItems, picks up to 5 DELIVERED
    items without evalScore (oldest-first), calls FlagEvalJudge with the ground-
    truth label from src/main/resources/eval/ground-truth-flags.jsonl, then
    calls ResearchItemEntity.recordEval(score, rationale) per item.

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with GET /research, GET /research/{id},
    POST /research/{id}/acknowledge (body {reviewedBy, notes?}),
    GET /research/sse, and /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/. acknowledge writes ItemArchived.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ResearchTasks.java declaring three Task<R> constants: CLASSIFY
  (ClassificationResult), SYNTHESISE (ResearchFlag), EVAL_FLAG (FlagEvalResult).
- Domain records ResearchItem, ClassificationResult, ResearchFlag, FlagEvalResult.
- Enum RelevanceCategory, ItemStatus.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9936
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/research-items.jsonl with 10 canned items
  covering CREDIT_RELEVANT (spread-widening note), RISK_RELEVANT (covenant filing),
  MARKET_COLOR (sector rotation commentary), and NOISE (press releases, duplicates).
- src/main/resources/eval/ground-truth-flags.jsonl with 6 labelled items
  (itemId, expectedCategory, rationale) for eval replay.
- src/main/resources/metadata/ containing copies of eval-matrix.yaml,
  risk-survey.yaml, README.md.
- eval-matrix.yaml at project root with 2 controls: G1 guardrail
  before-agent-response flag-completeness, E1 eval-periodic precision-recall.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at project root pre-filled for finance-analysis domain.
  decisions.authority_level = flag-only, oversight.human_in_loop = true,
  failure.failure_modes including "false-positive-flag-distracting-analyst" and
  "false-negative-missed-credit-event"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/relevance-classifier.md, prompts/synthesis.md, prompts/flag-eval-judge.md
  loaded as agent system prompts.
- README.md at the project root (already present in the blueprint folder).
- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs matching the formal exemplar. App UI tab: two-column
  layout (left = live feed with classification chips + status badges + REVIEW_REQUIRED
  banner; right = selected item detail with flag fields + eval score chip). Browser
  title exactly: <title>Akka Sample: Sector Market Researcher</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key (5 options: mock / env var /
  env file / secrets URI / type-once). NEVER write the key value to any file.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry via
  MockModelProvider.seedFor(itemId), and deserialises into the typed return.
- Per-agent mock-response shapes:
    relevance-classifier.json — 8–10 ClassificationResult entries spanning all four
      categories. Majority CREDIT_RELEVANT / RISK_RELEVANT for the seeded corpus
      so journeys complete quickly. At least 2 NOISE entries.
    synthesis.json — 6–8 ResearchFlag entries. Most have non-empty sourceReference
      and confidenceLevel (guardrail passes). At least 2 entries with blank
      sourceReference or blank confidenceLevel so the guardrail-block journey (J3)
      is reachable with a specific seed.
    flag-eval-judge.json — 6–8 FlagEvalResult entries with score 1–5,
      precisionEstimate 0.6–1.0, recallEstimate 0.5–1.0, one-sentence rationales.
- MockModelProvider.seedFor(itemId) makes selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 are load-bearing for
  this pattern.
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- The before-agent-response guardrail fires AFTER SynthesisAgent returns and BEFORE
  the flag is committed — not a prompt constraint, not a post-hoc view filter.
- The generated static-resources/index.html must include mermaid CSS overrides and
  theme variables from Lesson 24.
- Tab switching MUST match by data-tab / data-panel attribute, never by NodeList
  index (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build".
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
