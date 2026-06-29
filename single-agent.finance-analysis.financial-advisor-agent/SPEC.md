# SPEC — financial-advisor-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** FinancialAdvisorAgent.
**One-line pitch:** A client submits a portfolio snapshot and a natural-language investment question; one AI agent calls portfolio and market data tools and returns a structured advisory response — a recommendation, a risk rating, and a per-holding breakdown — after sector guardrails verify the response is within authorized scope.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `FinancialAdvisorAgent` (AutonomousAgent) carries the entire advisory decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **sector sanitizer** runs inside a Consumer between the raw client submission and the agent call — so the model never sees account numbers, tax identifiers, or national ID tokens that could leak regulated data into an LLM provider log.
- A **before-agent-response guardrail** validates the agent's advisory on every turn: all cited asset classes are in the authorized set for the client's account type, the risk rating is within the allowed enum, no speculative instruments are recommended without an explicit suitability waiver flag, and the response is parseable into `AdvisoryResponse` JSON. A non-conforming response triggers a retry inside the same task.

The blueprint shows that a single-agent advisory system can carry meaningful governance weight: two independent checks flank the one model call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **client profile** from a dropdown (or pastes a custom profile JSON) — the profile carries the account type (BROKERAGE, IRA, ROTH_IRA), a list of current holdings, and the client's risk tolerance label.
2. The user types an **investment question** in the question textarea (e.g., "Should I rebalance toward international equities given current valuations?").
3. The user clicks **Ask advisor**. The UI POSTs to `/api/advisories` and receives an `advisoryId`.
4. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted profile is visible in the card detail, listing which identifier categories were stripped.
5. Within ~10–30 s, the workflow's `adviseStep` completes. The card transitions to `ADVISING` then `RESPONSE_RECORDED`. The advisory appears: a risk rating badge (CONSERVATIVE / MODERATE / AGGRESSIVE), a short recommendation paragraph, and a per-holding breakdown table (ticker, current weight, suggested weight, rationale).
6. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdvisoryEndpoint` | `HttpEndpoint` | `/api/advisories/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AdvisoryEntity`, `AdvisoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AdvisoryEntity` | `EventSourcedEntity` | Per-advisory lifecycle: requested → sanitized → advising → response → audited. Source of truth. | `AdvisoryEndpoint`, `ClientDataSanitizer`, `AdvisoryWorkflow` | `AdvisoryView` |
| `ClientDataSanitizer` | `Consumer` | Subscribes to `AdvisoryRequested` events; strips regulated identifiers; calls `AdvisoryEntity.attachSanitized`. | `AdvisoryEntity` events | `AdvisoryEntity` |
| `AdvisoryWorkflow` | `Workflow` | One workflow per advisory. Steps: `awaitSanitizedStep` → `adviseStep` → `auditStep`. | started by `ClientDataSanitizer` once sanitized event lands | `FinancialAdvisorAgent`, `AdvisoryEntity` |
| `FinancialAdvisorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives client question and account type in task instructions; receives the sanitized portfolio snapshot as a task attachment; calls portfolio/market tools; returns `AdvisoryResponse`. | invoked by `AdvisoryWorkflow` | returns advisory |
| `AdvisoryView` | `View` | Read model: one row per advisory for the UI. | `AdvisoryEntity` events | `AdvisoryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Holding(
    String ticker,
    String assetClass,
    double currentWeight,
    double marketValue
) {}

record ClientProfile(
    String clientId,
    AccountType accountType,
    RiskTolerance riskTolerance,
    List<Holding> holdings,
    String submittedBy
) {}
enum AccountType { BROKERAGE, IRA, ROTH_IRA }
enum RiskTolerance { CONSERVATIVE, MODERATE, AGGRESSIVE }

record SanitizedProfile(
    String redactedProfileJson,
    List<String> identifierCategoriesFound
) {}

record HoldingAdvice(
    String ticker,
    String assetClass,
    double currentWeight,
    double suggestedWeight,
    String rationale
) {}

record AdvisoryResponse(
    RiskRating riskRating,
    String recommendation,
    List<HoldingAdvice> holdingAdvice,
    Instant advisedAt
) {}
enum RiskRating { CONSERVATIVE, MODERATE, AGGRESSIVE }

record AdvisoryRequest(
    String advisoryId,
    ClientProfile profile,
    String question,
    Instant requestedAt
) {}

record Advisory(
    String advisoryId,
    Optional<AdvisoryRequest> request,
    Optional<SanitizedProfile> sanitized,
    Optional<AdvisoryResponse> response,
    AdvisoryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AdvisoryStatus {
    REQUESTED, SANITIZED, ADVISING, RESPONSE_RECORDED, AUDITED, FAILED
}
```

Events on `AdvisoryEntity`: `AdvisoryRequested`, `ClientDataSanitized`, `AdvisingStarted`, `ResponseRecorded`, `AuditLogged`, `AdvisoryFailed`.

Every nullable lifecycle field on the `Advisory` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/advisories` — body `{ profile: ClientProfile, question: String, submittedBy: String }` → `{ advisoryId }`.
- `GET /api/advisories` — list all advisories, newest-first.
- `GET /api/advisories/{id}` — one advisory.
- `GET /api/advisories/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Financial Advisor Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted advisories (status pill + risk rating badge + age) and a right pane with the selected advisory's detail — the question, sanitized profile preview, recommendation paragraph, holding-advice table, and the audit timestamp.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Sector sanitizer** (`sector`, applied inside `ClientDataSanitizer` Consumer): strips account numbers (IBAN, brokerage account ID patterns), tax identifiers (SSN, EIN, TIN-like tokens), national IDs, and payment-card-like tokens from the raw client profile JSON before any LLM call. Records which identifier categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `FinancialAdvisorAgent`. Asserts (1) the candidate response is well-formed `AdvisoryResponse` JSON, (2) every `holdingAdvice[].assetClass` is within the authorized set for the account type (e.g., IRAs may not recommend crypto instruments without an explicit suitability-waiver flag), (3) the `riskRating` enum value is one of `{CONSERVATIVE, MODERATE, AGGRESSIVE}`, and (4) no holding in `holdingAdvice` has `suggestedWeight > 1.0` or `< 0.0`. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `FinancialAdvisorAgent` → `prompts/financial-advisor.md`. The single decision-making LLM. System prompt instructs it to read the sanitized portfolio snapshot, interpret the client's question in the context of their account type and risk tolerance, and return one `HoldingAdvice` per current holding plus a top-level recommendation.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the IRA seed client with a rebalancing question; within 30 s the advisory appears with one `HoldingAdvice` per holding and a risk rating badge.
2. **J2** — The agent's first response on a ROTH_IRA client recommends a crypto instrument (not in authorized set) — the `before-agent-response` guardrail rejects it; the second iteration produces a compliant advisory; the non-compliant draft never appears in the UI.
3. **J3** — A profile containing `SSN 987-65-4321` and `Account: BR-00012345` is submitted; the LLM call log shows only `[REDACTED-SSN]` and `[REDACTED-ACCOUNT]`; the entity's `request.profile` retains the raw fields for audit.
4. **J4** — A brokerage client with 10 holdings submits a diversification question; the advisory's `holdingAdvice` array contains exactly 10 entries, one per holding, each with a non-empty `rationale`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named financial-advisor-agent demonstrating the single-agent × finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-financial-advisor-agent. Java package
io.akka.samples.genaipoweredfinancialadvisortools. Akka 3.6.0. HTTP port 9699.

Components to wire (exactly):

- 1 AutonomousAgent FinancialAdvisorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/financial-advisor.md>) and
  .capability(TaskAcceptance.of(ADVISE_CLIENT).maxIterationsPerTask(3)). The task receives
  the client question and account type as instruction text and the sanitized portfolio
  snapshot JSON as a task ATTACHMENT (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  AdvisoryResponse{riskRating: RiskRating, recommendation: String,
  holdingAdvice: List<HoldingAdvice>, advisedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  3-iteration budget.

- 1 Workflow AdvisoryWorkflow per advisoryId with three steps:
  * awaitSanitizedStep — polls AdvisoryEntity.getAdvisory every 1s; on
    advisory.sanitized().isPresent() advances to adviseStep.
    WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * adviseStep — emits AdvisingStarted, then calls componentClient.forAutonomousAgent(
    FinancialAdvisorAgent.class, "advisor-" + advisoryId).runSingleTask(
      TaskDef.instructions(formatQuestion(advisory.request.question, advisory.request.profile.accountType()))
        .attachment("portfolio.json", advisory.sanitized.redactedProfileJson.getBytes())
    ) — returns a taskId, then forTask(taskId).result(ADVISE_CLIENT) to fetch the response.
    On success calls AdvisoryEntity.recordResponse(response). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(AdvisoryWorkflow::error).
  * auditStep — appends an immutable audit log entry to the entity carrying the advisoryId,
    advisedAt timestamp, riskRating, and a SHA-256 digest of the serialized
    AdvisoryResponse. No LLM call. Emits AuditLogged. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AdvisoryEntity (one per advisoryId). State
  Advisory{advisoryId: String, request: Optional<AdvisoryRequest>,
  sanitized: Optional<SanitizedProfile>, response: Optional<AdvisoryResponse>,
  status: AdvisoryStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  AdvisoryStatus enum: REQUESTED, SANITIZED, ADVISING, RESPONSE_RECORDED, AUDITED, FAILED.
  Events: AdvisoryRequested{request}, ClientDataSanitized{sanitized},
  AdvisingStarted{}, ResponseRecorded{response}, AuditLogged{digestHex},
  AdvisoryFailed{reason}. Commands: submit, attachSanitized, markAdvising, recordResponse,
  recordAudit, fail, getAdvisory. emptyState() returns Advisory.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer ClientDataSanitizer subscribed to AdvisoryEntity events; on AdvisoryRequested
  runs a regex+heuristic redaction pipeline (IBAN patterns, brokerage account IDs
  matching BR-\d{8,}, SSN-like, EIN-like, national-id-like, payment-card-like tokens) over
  the serialized ClientProfile JSON, computes the list of identifier categories found, builds
  SanitizedProfile, then calls AdvisoryEntity.attachSanitized(sanitized). After
  attachSanitized lands, the same Consumer starts an AdvisoryWorkflow with id = "advisory-"
  + advisoryId.

- 1 View AdvisoryView with row type AdvisoryRow (mirrors Advisory minus
  request.profile.clientId and the raw holding market values — the audit log retains those;
  the view holds the sanitized form for the UI). Table updater consumes AdvisoryEntity
  events. ONE query getAllAdvisories: SELECT * AS advisories FROM advisory_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * AdvisoryEndpoint at /api with POST /advisories (body
    {profile: {clientId, accountType, riskTolerance, holdings: [{ticker, assetClass,
    currentWeight, marketValue}], submittedBy}, question: String}; mints advisoryId;
    calls AdvisoryEntity.submit; returns {advisoryId}), GET /advisories (list from
    getAllAdvisories, sorted newest-first), GET /advisories/{id} (one row), GET
    /advisories/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AdvisoryTasks.java declaring one Task<R> constant: ADVISE_CLIENT = Task.name("Advise
  client").description("Read the attached sanitized portfolio and answer the client question
  with a structured AdvisoryResponse").resultConformsTo(AdvisoryResponse.class). DO NOT
  skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Holding, ClientProfile, AccountType, RiskTolerance, SanitizedProfile,
  HoldingAdvice, AdvisoryResponse, RiskRating, AdvisoryRequest, Advisory, AdvisoryStatus.

- RecommendationGuardrail.java implementing the before-agent-response hook. Reads the
  candidate AdvisoryResponse from the LLM response, runs the four checks listed in
  eval-matrix.yaml G1, and either passes the response through or returns
  Guardrail.reject(<structured-error>) to force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9699 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The FinancialAdvisorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-clients.jsonl with 3 seeded client profiles:
  a conservative IRA holder with 5 equity + 3 bond holdings,
  a moderate BROKERAGE holder with 8 diversified holdings,
  an aggressive ROTH_IRA holder with 6 growth-equity holdings.
  Each profile contains 1–2 plausible regulated identifiers so S1 has work to do.

- src/main/resources/sample-events/seed-questions.jsonl with 3 paired example questions,
  one per seeded client profile (rebalancing, income vs. growth, sector-tilt question).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. regulation_anchors: [] until
  the deployer confirms jurisdictional scope.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  sector = finance, decisions.authority_level = recommend-only (the agent's advisory is
  advisory, not a binding order), oversight.human_in_loop = true (a human advisor or the
  client reads the response before acting), failure.failure_modes including
  "unauthorized-asset-class", "hallucinated-ticker", "weight-exceeds-bounds",
  "regulated-id-leakage", "suitability-mismatch"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/financial-advisor.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Financial Advisor Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of advisory cards; right = selected-advisory detail with client question,
  sanitized profile preview, recommendation paragraph, holding-advice table, and audit
  timestamp). Browser title exactly:
  <title>Akka Sample: Financial Advisor Agent</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(advisoryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    advise-client.json — 8 AdvisoryResponse entries covering all three RiskRating values.
      Each entry has a recommendation paragraph and a holdingAdvice array sized to the
      seeded client's holding count. Each HoldingAdvice has a non-empty rationale, a
      suggestedWeight between 0.0 and 1.0, and an assetClass drawn from the authorized
      set for the relevant account type. Plus 2 deliberately NON-CONFORMING entries — one
      recommending a crypto instrument for an IRA (blocked by the asset-class check in G1),
      one with a suggestedWeight of 1.5 (blocked by the bounds check in G1) — so J2 is
      reproducible. The mock selects a non-conforming entry on the FIRST iteration of every
      3rd advisory (modulo seed).
- A MockModelProvider.seedFor(advisoryId) helper makes per-advisory selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FinancialAdvisorAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion AdvisoryTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (adviseStep
  60s, awaitSanitizedStep 15s, auditStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Advisory row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: AdvisoryTasks.java with ADVISE_CLIENT = Task.name(...).description(...)
  .resultConformsTo(AdvisoryResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9699 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (FinancialAdvisorAgent). The audit step is deterministic (SHA-256 digest computation) and
  does NOT make an LLM call — keeping the pattern's "one agent" promise honest.
- The portfolio snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated adviseStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
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
