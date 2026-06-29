# SPEC — deal-strategy-analyst

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** DealStrategyAnalyst.
**One-line pitch:** A sales rep submits a deal context bundle (emails, call notes, CRM snapshot); one AI agent reads the bundle (passed as a task attachment, never as inline prompt text) and returns a structured `StrategyRecommendation` — prioritised next steps each with an action type, target stakeholder, reasoning, and deadline suggestion.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the sales-marketing domain. One `DealStrategyAgent` (AutonomousAgent) carries the entire analysis; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw deal context submission and the agent call — so the model never sees buyer email addresses, direct-dial phone numbers, or confidential company names where NDA restrictions apply.
- A **before-agent-response guardrail** validates the agent's recommendation on every turn: well-formed JSON, every next-step references a valid stakeholder role from the submitted context, every action type is in the allowed set, and the priority ranking is complete. A malformed recommendation triggers a retry inside the same task.

The blueprint also wires an **on-decision eval** that runs immediately after each `RecommendationRecorded` event, scoring the recommendation for actionability: are there concrete deadlines? Does each step address a named concern from the context? Is the priority ordering coherent with the deal signals provided?

The blueprint shows that the single-agent pattern does not mean "ungoverned" — independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes their deal context into the **Context** textarea (or picks one of three seeded examples — an enterprise software evaluation deal, a renewal at risk, a new logo in competitive selection).
2. The user picks a **deal type** from a dropdown (enterprise-new, renewal-risk, competitive) or enters a free-form deal name.
3. The user clicks **Analyse deal**. The UI POSTs to `/api/deals` and receives a `dealId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted context is visible in the card detail, with a small list of categories the sanitizer found.
5. Within ~10–30 s, the workflow's `analyseStep` completes. The card transitions to `ANALYSING` then `RECOMMENDATION_RECORDED`. The recommendation appears: a top-level priority badge (CRITICAL / HIGH / NORMAL), a short executive summary, and a ranked list of next steps (action type, stakeholder role, rationale, suggested deadline).
6. Within ~1 s of the recommendation, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the recommendation is sufficiently actionable.
7. The user can submit another deal; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DealEndpoint` | `HttpEndpoint` | `/api/deals/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `DealEntity`, `DealView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DealEntity` | `EventSourcedEntity` | Per-deal lifecycle: submitted → sanitized → analysing → recommendation → eval. Source of truth. | `DealEndpoint`, `DealContextSanitizer`, `AnalysisWorkflow` | `DealView` |
| `DealContextSanitizer` | `Consumer` | Subscribes to `DealSubmitted` events; redacts PII and confidential identifiers; calls `DealEntity.attachSanitized`. | `DealEntity` events | `DealEntity` |
| `AnalysisWorkflow` | `Workflow` | One workflow per deal. Steps: `awaitSanitizedStep` → `analyseStep` → `evalStep`. | started by `DealContextSanitizer` once sanitized event lands | `DealStrategyAgent`, `DealEntity` |
| `DealStrategyAgent` | `AutonomousAgent` | The one decision-making LLM. Receives deal type and rep context in the task definition and the sanitized context bundle as a task attachment; returns `StrategyRecommendation`. | invoked by `AnalysisWorkflow` | returns recommendation |
| `DealView` | `View` | Read model: one row per deal for the UI. | `DealEntity` events | `DealEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record StakeholderSignal(
    String role,          // e.g. "Economic Buyer", "Champion", "Legal"
    String concern,       // free-text concern the contact has raised
    String engagementLevel // "high", "medium", "low", "unknown"
) {}

record DealContext(
    String dealId,
    String dealName,
    String dealType,      // enterprise-new | renewal-risk | competitive
    String rawContext,    // emails + call notes + CRM snapshot blob
    List<StakeholderSignal> stakeholders,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedContext(
    String redactedContext,
    List<String> piiCategoriesFound
) {}

record NextStep(
    String stepId,
    ActionType actionType,
    String stakeholderRole,
    String rationale,
    String suggestedDeadline,  // ISO-8601 date or relative, e.g. "within 3 business days"
    int priority               // 1 = highest
) {}
enum ActionType { FOLLOW_UP_EMAIL, SCHEDULE_CALL, SEND_PROPOSAL, REQUEST_INTRO, ESCALATE, ADDRESS_OBJECTION, PROVIDE_REFERENCE }

record StrategyRecommendation(
    Urgency urgency,
    String executiveSummary,
    List<NextStep> nextSteps,
    Instant decidedAt
) {}
enum Urgency { CRITICAL, HIGH, NORMAL }

record EvalResult(
    int score,          // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Deal(
    String dealId,
    Optional<DealContext> context,
    Optional<SanitizedContext> sanitized,
    Optional<StrategyRecommendation> recommendation,
    Optional<EvalResult> eval,
    DealStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DealStatus {
    SUBMITTED, SANITIZED, ANALYSING, RECOMMENDATION_RECORDED, EVALUATED, FAILED
}
```

Events on `DealEntity`: `DealSubmitted`, `ContextSanitized`, `AnalysisStarted`, `RecommendationRecorded`, `EvaluationScored`, `DealFailed`.

Every nullable lifecycle field on the `Deal` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/deals` — body `{ dealName, dealType, rawContext, stakeholders: [StakeholderSignal], submittedBy }` → `{ dealId }`.
- `GET /api/deals` — list all deals, newest-first.
- `GET /api/deals/{id}` — one deal.
- `GET /api/deals/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: DealStrategyAnalyst</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted deals (status pill + urgency badge + age) and a right pane with the selected deal's detail — submitted stakeholder signals, sanitized context preview, recommendation summary, ranked next-step list, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `DealContextSanitizer` Consumer): redacts emails, phone numbers, company names where marked confidential, postal addresses, and account-id-like tokens from the raw deal context before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `DealStrategyAgent`. Asserts the candidate response is well-formed `StrategyRecommendation` JSON, every `nextSteps[].stakeholderRole` matches a role present in the submitted `stakeholders` list, every `actionType` is in `{FOLLOW_UP_EMAIL, SCHEDULE_CALL, SEND_PROPOSAL, REQUEST_INTRO, ESCALATE, ADDRESS_OBJECTION, PROVIDE_REFERENCE}`, and no two `nextSteps` share the same `priority` value. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `RecommendationRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that every next step has a non-empty `suggestedDeadline` and `rationale`, that `rationale` strings are longer than a single word, that the priority list has no gaps, and that the urgency is consistent with the highest-priority action type (a `CRITICAL` deal with only `FOLLOW_UP_EMAIL` steps loses a point). Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `DealStrategyAgent` → `prompts/deal-strategy-analyst.md`. The single decision-making LLM. System prompt instructs it to read the attached context bundle, identify the deal's most pressing risks and opportunities, and return a ranked list of next steps as a `StrategyRecommendation`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the enterprise-new seed deal; within 30 s the recommendation appears with at least three next steps, each with a non-empty `rationale`, `suggestedDeadline`, and `stakeholderRole`, plus an eval score chip.
2. **J2** — The agent's first response on a deal is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed recommendation; the UI never displays the malformed response.
3. **J3** — A recommendation whose next steps all have empty `suggestedDeadline` strings receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A deal context containing `sarah.johnson@buyer.com` and `+1-555-867-5309` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `context.rawContext` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named deal-strategy-analyst demonstrating the single-agent × sales-marketing cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-sales-marketing-deal-strategy-analyst. Java package io.akka.samples.dealstrategy.
Akka 3.6.0. HTTP port 9706.

Components to wire (exactly):

- 1 AutonomousAgent DealStrategyAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/deal-strategy-analyst.md>) and
  .capability(TaskAcceptance.of(ANALYSE_DEAL).maxIterationsPerTask(3)). The task receives
  deal type and stakeholder signals as its instruction text and the sanitized context bundle
  as a task ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name,
  contentBytes) is the canonical call). Output: StrategyRecommendation{urgency: Urgency
  (CRITICAL/HIGH/NORMAL), executiveSummary: String, nextSteps: List<NextStep>,
  decidedAt: Instant}. The agent is configured with a before-agent-response guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries the response within its 3-iteration budget.

- 1 Workflow AnalysisWorkflow per dealId with three steps:
  * awaitSanitizedStep — polls DealEntity.getDeal every 1s; on deal.sanitized().isPresent()
    advances to analyseStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * analyseStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    DealStrategyAgent.class, "analyst-" + dealId).runSingleTask(
      TaskDef.instructions(formatDealBrief(deal.context.dealType, deal.context.stakeholders))
        .attachment("deal-context.txt", deal.sanitized.redactedContext.getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANALYSE_DEAL) to fetch the recommendation.
    On success calls DealEntity.recordRecommendation(recommendation). WorkflowSettings
    .stepTimeout 60s with defaultStepRecovery maxRetries(2).failoverTo(AnalysisWorkflow::error).
  * evalStep — runs a deterministic rule-based RecommendationScorer (NOT an LLM call) over the
    recorded recommendation: checks that every NextStep has non-empty suggestedDeadline and
    rationale, that rationale is longer than one word, that priority values form a contiguous
    sequence with no gaps, and that urgency is consistent with action types. Emits
    EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity DealEntity (one per dealId). State Deal{dealId: String,
  context: Optional<DealContext>, sanitized: Optional<SanitizedContext>,
  recommendation: Optional<StrategyRecommendation>, eval: Optional<EvalResult>,
  status: DealStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  DealStatus enum: SUBMITTED, SANITIZED, ANALYSING, RECOMMENDATION_RECORDED, EVALUATED, FAILED.
  Events: DealSubmitted{context}, ContextSanitized{sanitized}, AnalysisStarted{},
  RecommendationRecorded{recommendation}, EvaluationScored{eval}, DealFailed{reason}.
  Commands: submit, attachSanitized, markAnalysing, recordRecommendation, recordEvaluation,
  fail, getDeal. emptyState() returns Deal.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer DealContextSanitizer subscribed to DealEntity events; on DealSubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, postal addresses,
  person-name heuristic, confidential-company-name tokens that appear in all-caps or
  in "NDA: …" patterns, account-id-like tokens) over rawContext, computes the list of
  categories found, builds SanitizedContext, then calls DealEntity.attachSanitized(sanitized).
  After attachSanitized lands, the same Consumer starts an AnalysisWorkflow with id =
  "analysis-" + dealId.

- 1 View DealView with row type DealRow (mirrors Deal minus context.rawContext — the audit
  log keeps the raw; the view holds the sanitized form for the UI). Table updater consumes
  DealEntity events. ONE query getAllDeals: SELECT * AS deals FROM deal_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * DealEndpoint at /api with POST /deals (body
    {dealName, dealType, rawContext, stakeholders: [{role, concern, engagementLevel}],
    submittedBy}; mints dealId; calls DealEntity.submit; returns {dealId}), GET /deals
    (list from getAllDeals, sorted newest-first), GET /deals/{id} (one row), GET
    /deals/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- DealTasks.java declaring one Task<R> constant: ANALYSE_DEAL = Task.name("Analyse deal")
  .description("Read the attached deal context bundle and produce a StrategyRecommendation
  with prioritised next steps").resultConformsTo(StrategyRecommendation.class). DO NOT skip
  this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records StakeholderSignal, DealContext, SanitizedContext, NextStep, ActionType,
  StrategyRecommendation, Urgency, EvalResult, Deal, DealStatus.

- RecommendationGuardrail.java implementing the before-agent-response hook. Reads the
  candidate StrategyRecommendation from the LLM response, runs the four checks listed in
  eval-matrix.yaml G1, and either passes the response through or returns
  Guardrail.reject(<structured-error>) to force the agent loop to retry.

- RecommendationScorer.java — pure deterministic logic (no LLM). Inputs: StrategyRecommendation
  and the submitted List<StakeholderSignal>. Outputs: EvalResult. Scoring rubric documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9706 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The DealStrategyAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-deals.jsonl with 3 seeded deal contexts:
  an enterprise-new deal at a 2000-seat financial services firm (4 stakeholders, 1800-word
  context with call notes and email threads), a renewal-risk deal where the champion
  has left (3 stakeholders, 1200-word context), and a competitive-selection deal against
  two incumbents (5 stakeholders, 2000-word context). Each contains 2–3 plausible PII
  strings (buyer email, phone) so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) and 1 eval (E1) matching
  the mechanisms in Section 8 of this SPEC. Matching simplified_view list.
  No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only
  (the agent's recommendation is advisory, not enforced), oversight.human_in_loop = true
  (a rep reads the recommendation before acting on it), failure.failure_modes including
  "hallucinated-stakeholder", "missed-concern", "pii-leakage-via-llm",
  "urgency-miscalibration", "actionless-recommendation"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/deal-strategy-analyst.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Deal Strategy Analyst", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of deal cards; right = selected-deal detail with submitted stakeholder signals,
  sanitized context preview, recommendation summary, ranked next-step list, and eval-score chip).
  Browser title exactly: <title>Akka Sample: DealStrategyAnalyst</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(dealId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    analyse-deal.json — 8 StrategyRecommendation entries covering all three Urgency values.
      Each entry has an executiveSummary paragraph and a nextSteps array with 3–5 NextStep
      items. Each NextStep has a non-empty stakeholderRole matching the seeded deal's
      stakeholder list, a non-empty rationale (2+ words), a suggestedDeadline (date or
      relative string), and a valid ActionType. Urgencies and action type mixes vary
      realistically across entries. Plus 2 deliberately MALFORMED entries (one with a
      nextSteps[].stakeholderRole not present in the submitted stakeholders list; one with
      an actionType value outside the enum) — the guardrail blocks both, exercising the retry
      path. The mock should select a malformed entry on the FIRST iteration of every 3rd deal
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(dealId) helper makes per-deal selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DealStrategyAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DealTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (analyseStep
  60s, awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Deal row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: DealTasks.java with ANALYSE_DEAL = Task.name(...).description(...)
  .resultConformsTo(StrategyRecommendation.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9706 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (DealStrategyAgent).
  The on-decision eval is rule-based (RecommendationScorer.java) and does NOT make an
  LLM call — keeping the pattern's "one agent" promise honest.
- The deal context is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated analyseStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
