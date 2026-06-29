# SPEC — sba-loan-processor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Small Business Loan Agent.
**One-line pitch:** A loan officer submits a business loan application; one `LoanUnderwritingAgent` walks it through four task phases — **INTAKE** applicant data, **UNDERWRITE** the financials, **DECIDE** on a recommendation, **REPORT** a structured underwriting memo — with PII and protected attributes sanitized before the agent sees them, a human loan officer approving or denying every decision, and a fair-lending evaluator scoring every recommendation.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a small-business finance domain. One `LoanUnderwritingAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the INTAKE task's typed output becomes the UNDERWRITE task's instruction context; the UNDERWRITE task's output becomes the DECISION task's context; the DECISION task's output becomes the REPORT task's context. The agent never holds all four phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Four governance mechanisms are wired around the pipeline:

- A **PII sanitizer** (`before-call`) strips direct applicant identifiers — SSN, full legal name, date of birth, home address — from the agent's input context before each task. The agent reasons about financial characteristics only; it never sees the identifier fields. The sanitized fields are stored on `LoanApplicationEntity` under a separate `sensitiveData` field and returned to authorized humans only.
- A **special-category sanitizer** (`before-call`) masks protected attributes — race, ethnicity, sex, age, marital status, national origin — from the applicant record before the agent's instruction context is assembled. The underwriting analysis cannot reference these fields; the fair-lending scorer checks the output for any proxy language.
- A **human-in-the-loop (HITL) gate** holds the workflow in `PENDING_REVIEW` after `DecisionMade` lands. A loan officer opens the officer queue, reviews the underwriting memo and the agent's structured recommendation, and submits an `approve` or `deny` decision. The workflow resumes only after the officer's action.
- An **`on-decision-eval`** runs immediately after `DecisionMade` lands, as `evalStep` inside the workflow. A deterministic `FairLendingScorer` (no LLM call — the eval is rule-based on purpose) checks four signals: denial-rate consistency (same tier, same period, no protected-class clustering), rationale completeness (every denial cites at least one financial criterion), absence of proxy language in the rationale, and debt-service-coverage ratio (DSCR) calculation correctness.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the sanitizers enforce input hygiene, the phase-gate enforces task ordering, the HITL gate enforces human accountability, and the on-decision eval surfaces fairness signals, all independently.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **loan application** from the seeded list (or uploads a custom JSON) and clicks **Process application**. The UI POSTs to `/api/loans` and receives an `applicationId`.
2. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `INTAKE_IN_PROGRESS` — the workflow has started `intakeStep` and the agent has been handed the INTAKE task with PII already sanitized.
3. Within ~15–25 s the card reaches `UNDERWRITING` — the typed `CreditProfile` is visible in the card detail (credit score tier, cash-flow summary, collateral estimate). The agent's INTAKE task returned; the workflow recorded `IntakeCompleted` and ran the UNDERWRITE task.
4. Within ~15–25 s more the card reaches `DECISION_IN_PROGRESS`. The `UnderwritingAnalysis` is visible (risk tier, key risk factors, mitigants).
5. Within ~15–25 s more the card reaches `PENDING_REVIEW`. The right pane shows the full typed `LoanDecision` — recommendation (APPROVE / APPROVE_WITH_CONDITIONS / DENY), proposed amount, interest-rate tier, decision rationale, conditions list — plus the fair-lending eval score chip (1–5).
6. A loan officer opens the officer queue tab, reads the memo, and clicks **Approve** or **Deny**. The entity transitions to `OFFICER_APPROVED` or `OFFICER_DENIED` and the SSE stream updates the live list.
7. The user can submit another application; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `LoanEndpoint` | `HttpEndpoint` | `/api/loans/*` — submit, list, get, SSE, officer review; serves `/api/metadata/*`. | — | `LoanApplicationEntity`, `LoanApplicationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `LoanApplicationEntity` | `EventSourcedEntity` | Per-application lifecycle: submitted → intake → underwriting → decision → pending_review → officer_approved / officer_denied / failed. Source of truth; stores sanitized agent context and raw sensitive data under separate fields. | `LoanEndpoint`, `LoanPipelineWorkflow` | `LoanApplicationView` |
| `LoanPipelineWorkflow` | `Workflow` | One workflow per application. Steps: `intakeStep` → `underwriteStep` → `decisionStep` → `hitlStep` → `evalStep` → `reportStep`. Each agent step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `hitlStep` is a pause step — the workflow suspends until `LoanEndpoint` delivers an officer decision. | started by `LoanEndpoint` after `SUBMITTED` | `LoanUnderwritingAgent`, `LoanApplicationEntity` |
| `LoanUnderwritingAgent` | `AutonomousAgent` | The single agent. Declares four `Task<R>` constants in `LoanTasks.java`: `INTAKE_APPLICATION` → `CreditProfile`, `UNDERWRITE_APPLICATION` → `UnderwritingAnalysis`, `MAKE_DECISION` → `LoanDecision`, `GENERATE_REPORT` → `UnderwritingMemo`. Each task is registered with the phase-appropriate function tools. | invoked by `LoanPipelineWorkflow` | returns typed results |
| `IntakeTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `lookupCreditScore(businessEin)` and `fetchBusinessProfile(businessEin)`. Reads from `src/main/resources/sample-data/applications/*.json` for deterministic offline output. | called from INTAKE task | returns `CreditScore`, `BusinessProfile` |
| `UnderwriteTools` | function-tools class | Implements `analyzeCashFlow(financials)` and `valuateCollateral(collateral)`. In-memory deterministic calculations. | called from UNDERWRITE task | returns `CashFlowSummary`, `CollateralValue` |
| `DecisionTools` | function-tools class | Implements `computeDscr(cashFlow, debtObligations)` and `classifyRiskTier(creditScore, dscr, collateral)`. | called from DECISION task | returns `DscrResult`, `RiskTier` |
| `ReportTools` | function-tools class | Implements `formatDecisionRationale(analysis, decision)` and `compileConditions(riskTier, decision)`. | called from REPORT task | returns `RationaleText`, `List<LoanCondition>` |
| `PiiSanitizer` | `before-call` sanitizer (registered on `LoanUnderwritingAgent`) | Strips SSN, full legal name, date of birth, and home address from the applicant record before the agent's context is assembled. The agent receives a sanitized view; the entity retains the originals under `sensitiveData`. | every agent task on every application | sanitized context / pass-through |
| `ProtectedAttributeSanitizer` | `before-call` sanitizer (registered on `LoanUnderwritingAgent`) | Masks race, ethnicity, sex, age, marital status, and national origin from the applicant record before the UNDERWRITE and DECISION tasks. The scorer checks the output for proxy language. | every UNDERWRITE / DECISION task | masked context |
| `PhaseGuardrail` | `before-tool-call` guardrail (registered on `LoanUnderwritingAgent`) | Reads the in-flight task's declared phase and the current `LoanApplicationEntity.status`. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `FairLendingScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `LoanDecision`, `UnderwritingAnalysis`, `CreditProfile`. Output: `FairnessEvalResult{score, flags, rationale}`. | called from `evalStep` | returns score |
| `LoanApplicationView` | `View` | Read model: one row per application for the UI. | `LoanApplicationEntity` events | `LoanEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BusinessProfile(
    String businessEin,           // masked in agent context by PiiSanitizer
    String businessName,
    String industry,
    int yearsInOperation,
    String legalStructure
) {}

record CreditScore(
    String tier,                  // EXCELLENT / GOOD / FAIR / POOR
    int score,
    Instant fetchedAt
) {}

record CreditProfile(
    CreditScore creditScore,
    BusinessProfile businessProfile,
    Instant profiledAt
) {}

record CashFlowSummary(
    BigDecimal annualRevenue,
    BigDecimal operatingExpenses,
    BigDecimal netOperatingIncome,
    Instant analyzedAt
) {}

record CollateralValue(
    String collateralType,
    BigDecimal estimatedValue,
    BigDecimal ltvRatio
) {}

record UnderwritingAnalysis(
    CashFlowSummary cashFlow,
    CollateralValue collateral,
    List<String> keyRiskFactors,
    List<String> mitigants,
    Instant analyzedAt
) {}

record DscrResult(
    BigDecimal dscr,
    BigDecimal annualDebtService,
    boolean meetsThreshold          // threshold: DSCR >= 1.25
) {}

enum RiskTier { TIER_1, TIER_2, TIER_3, TIER_4 }

enum RecommendationType { APPROVE, APPROVE_WITH_CONDITIONS, DENY }

record LoanCondition(String conditionId, String description, String conditionType) {}

record LoanDecision(
    RecommendationType recommendation,
    BigDecimal proposedAmount,
    String interestRateTier,
    String rationale,
    List<LoanCondition> conditions,
    DscrResult dscrResult,
    RiskTier riskTier,
    Instant decidedAt
) {}

record RationaleText(String rationale) {}

record UnderwritingMemo(
    String applicationId,
    CreditProfile creditProfile,
    UnderwritingAnalysis analysis,
    LoanDecision decision,
    FairnessEvalResult fairnessEval,
    Instant generatedAt
) {}

record FairnessEvalResult(
    int score,                     // 1..5
    List<String> flags,
    String rationale,
    Instant evaluatedAt
) {}

record OfficerReview(
    String officerId,
    String reviewOutcome,           // APPROVED / DENIED
    String officerNotes,
    Instant reviewedAt
) {}

record LoanApplicationRecord(
    String applicationId,
    Optional<String> applicantRef,  // masked reference; PII in sensitiveData
    Optional<CreditProfile> creditProfile,
    Optional<UnderwritingAnalysis> analysis,
    Optional<LoanDecision> decision,
    Optional<FairnessEvalResult> fairnessEval,
    Optional<UnderwritingMemo> memo,
    Optional<OfficerReview> officerReview,
    LoanApplicationStatus status,
    Instant submittedAt,
    Optional<Instant> finishedAt
) {}

enum LoanApplicationStatus {
    SUBMITTED, INTAKE_IN_PROGRESS, INTAKE_COMPLETE,
    UNDERWRITING, UNDERWRITING_COMPLETE,
    DECISION_IN_PROGRESS, DECISION_MADE,
    PENDING_REVIEW, OFFICER_APPROVED, OFFICER_DENIED, FAILED
}
```

Events on `LoanApplicationEntity`: `ApplicationSubmitted`, `IntakeStarted`, `IntakeCompleted`, `UnderwriteStarted`, `UnderwritingCompleted`, `DecisionStarted`, `DecisionMade`, `FairnessEvaluated`, `MemoGenerated`, `OfficerReviewRequested`, `OfficerApproved`, `OfficerDenied`, `GuardrailRejected`, `ApplicationFailed`.

Every nullable lifecycle field on the `LoanApplicationRecord` state record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/loans` — body `{ applicantRef, loanAmount, loanPurpose }` → `{ applicationId }`.
- `GET /api/loans` — list all applications, newest-first.
- `GET /api/loans/{id}` — one application.
- `GET /api/loans/sse` — Server-Sent Events; one event per state transition.
- `POST /api/loans/{id}/review` — officer submits `{ officerId, outcome, notes }` → `200`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Small Business Loan Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of applications (status pill + applicant reference + age + eval score chip) and a right pane with the selected application's detail — credit profile, underwriting analysis, loan decision, fairness eval score, officer review controls (when status is `PENDING_REVIEW`), and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer (`before-call`)**: `PiiSanitizer` is registered on `LoanUnderwritingAgent` and runs before the agent's context is assembled for every task. It reads the `LoanApplicationRecord.sensitiveData` map and removes `ssn`, `fullLegalName`, `dateOfBirth`, and `homeAddress` from the fields passed to the agent. The sanitized output is logged at `DEBUG` with the field names only (never values). The sensitive fields remain on the entity under `sensitiveData` and are returned to authorized API callers only via a separate `/api/loans/{id}/sensitive` endpoint protected by an officer-role check.
- **S2 — Protected-attribute sanitizer (`before-call`)**: `ProtectedAttributeSanitizer` is registered on `LoanUnderwritingAgent` and masks `race`, `ethnicity`, `sex`, `age`, `maritalStatus`, and `nationalOrigin` from the applicant record before the UNDERWRITE and DECISION task contexts are assembled. Masking replaces each value with a `[REDACTED]` sentinel so the agent cannot infer the original value. The `FairLendingScorer` checks the resulting `LoanDecision.rationale` for proxy terms using a hard-coded term list.
- **H1 — Human-in-the-loop gate**: `LoanPipelineWorkflow.hitlStep` suspends the workflow after `DecisionMade` lands on the entity, setting status to `PENDING_REVIEW` and emitting `OfficerReviewRequested`. The workflow remains paused indefinitely until `LoanEndpoint` receives a `POST /api/loans/{id}/review` call from an authenticated loan officer. On receipt, the workflow resumes: if `outcome = APPROVED` it emits `OfficerApproved` and advances to `reportStep`; if `outcome = DENIED` it emits `OfficerDenied` and ends. No application can reach `OFFICER_APPROVED` or `OFFICER_DENIED` without a human decision.
- **E1 — `on-decision-eval`**: runs immediately after `DecisionMade` lands, as `evalStep` inside the workflow. `FairLendingScorer` is deterministic and rule-based (no LLM call — keeping the single-agent pipeline invariant honest). Four checks, one point per check satisfied on a base of 1: (1) denial-rate pattern — the running denial count in the same risk tier does not exceed 2× the overall approval rate for that tier in the seeded corpus; (2) rationale completeness — every `DENY` or `APPROVE_WITH_CONDITIONS` decision cites at least one financial criterion from the accepted list; (3) proxy-language absence — the rationale contains none of the terms on the `FairLendingScorer.PROXY_TERMS` list; (4) DSCR correctness — the `dscrResult.dscr` value equals `netOperatingIncome / annualDebtService` within a 0.01 tolerance. Emits `FairnessEvaluated{score: 1-5, flags, rationale}`.

## 9. Agent prompts

- `LoanUnderwritingAgent` → `prompts/loan-underwriting-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat the sanitized applicant record as the entire context for each phase, and return the task's typed output without referencing any protected attributes or personal identifiers.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — A loan officer submits the seeded application `app-001` (Sunrise Bakery, 7(a) $250,000); within 90 s the application reaches `PENDING_REVIEW` with a non-null `CreditProfile`, `UnderwritingAnalysis`, `LoanDecision`, and a fairness eval score chip on the card. The officer approves; within 2 s the card reaches `OFFICER_APPROVED`.
2. **J2** — The mock LLM's first iteration on an INTAKE task calls `computeDscr` (a DECISION-phase tool) before `UnderwritingCompleted` has been recorded. `PhaseGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the application eventually reaches `PENDING_REVIEW`. The UI's rejection-log strip shows the one rejected call.
3. **J3** — The raw applicant record for `app-004` includes `race` and `maritalStatus` fields. The underwriting analysis and decision rationale stored on the entity contain neither of those values nor any proxy term from the `PROXY_TERMS` list.
4. **J4** — The mock's `write-decision.json` includes one entry whose `rationale` contains the proxy term `neighborhood stability`. `FairLendingScorer` flags it; the card's eval score chip shows ≤ 2 with a rationale naming the flagged term.
5. **J5** — An officer submits a `deny` review for `app-002`; the entity transitions to `OFFICER_DENIED`; no `UnderwritingMemo` is generated; the SSE stream updates all listening clients.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sba-loan-processor demonstrating the sequential-pipeline x
finance-analysis cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-finance-analysis-sba-loan-processor.
Java package io.akka.samples.smallbusinessloanagent. Akka 3.6.0. HTTP port 9999.

Components to wire (exactly):

- 1 AutonomousAgent LoanUnderwritingAgent — extends
  akka.javasdk.agent.autonomous.AutonomousAgent. definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/loan-underwriting-agent.md>) and four
  .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(4)) entries — one per declared Task.
  Function tools are registered with .tools(...) — IntakeTools, UnderwriteTools, DecisionTools,
  and ReportTools are ALL registered on the agent; phase gating is the job of PhaseGuardrail,
  NOT of conditional .tools(...) wiring. Two before-call sanitizers (PiiSanitizer,
  ProtectedAttributeSanitizer) and one before-tool-call guardrail (PhaseGuardrail) are
  registered on the agent via the agent's guardrail/sanitizer configuration block. On guardrail
  rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow LoanPipelineWorkflow per applicationId with six steps:
  * intakeStep — emits IntakeStarted on the entity, then calls componentClient
    .forAutonomousAgent(LoanUnderwritingAgent.class, "agent-" + applicationId).runSingleTask(
      TaskDef.instructions("ApplicationId: " + applicationId + "\nPhase: INTAKE\nSanitized
      applicant: " + sanitizedJson)
        .metadata("applicationId", applicationId)
        .metadata("phase", "INTAKE")
        .taskType(LoanTasks.INTAKE_APPLICATION)
    ). Reads forTask(taskId).result(INTAKE_APPLICATION) to get CreditProfile. Writes
    LoanApplicationEntity.recordCreditProfile(creditProfile). WorkflowSettings.stepTimeout 60s.
  * underwriteStep — emits UnderwriteStarted, then runSingleTask with instructions
    (formatUnderwriteContext(creditProfile)) and metadata.phase = "UNDERWRITE", taskType
    UNDERWRITE_APPLICATION. Writes LoanApplicationEntity.recordAnalysis(analysis). stepTimeout 60s.
  * decisionStep — emits DecisionStarted, then runSingleTask with instructions
    (formatDecisionContext(analysis, creditProfile)) and metadata.phase = "DECISION", taskType
    MAKE_DECISION. Writes LoanApplicationEntity.recordDecision(decision). stepTimeout 60s.
  * hitlStep — emits OfficerReviewRequested, sets status PENDING_REVIEW. Workflow PAUSES.
    Resumed by LoanEndpoint.submitOfficerReview(applicationId, officerReview). stepTimeout
    indefinite (no timeout — HITL gates have no time bound). On resume: if officerReview
    .reviewOutcome = "APPROVED" → advance to evalStep; if "DENIED" → emit OfficerDenied → end.
  * evalStep — runs FairLendingScorer over (decision, analysis, creditProfile) and writes
    LoanApplicationEntity.recordFairnessEval(fairnessEval). stepTimeout 5s.
  * reportStep — emits MemoGenerated after runSingleTask(GENERATE_REPORT). stepTimeout 60s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(LoanPipelineWorkflow::error). The error step writes
  ApplicationFailed and ends.

- 1 EventSourcedEntity LoanApplicationEntity (one per applicationId). State
  LoanApplicationRecord{applicationId, applicantRef: Optional<String>, creditProfile:
  Optional<CreditProfile>, analysis: Optional<UnderwritingAnalysis>, decision:
  Optional<LoanDecision>, fairnessEval: Optional<FairnessEvalResult>, memo:
  Optional<UnderwritingMemo>, officerReview: Optional<OfficerReview>, status:
  LoanApplicationStatus, submittedAt: Instant, finishedAt: Optional<Instant>}.
  LoanApplicationStatus enum: SUBMITTED, INTAKE_IN_PROGRESS, INTAKE_COMPLETE, UNDERWRITING,
  UNDERWRITING_COMPLETE, DECISION_IN_PROGRESS, DECISION_MADE, PENDING_REVIEW,
  OFFICER_APPROVED, OFFICER_DENIED, FAILED. Events: ApplicationSubmitted{applicantRef,
  loanAmount, loanPurpose}, IntakeStarted, IntakeCompleted{creditProfile}, UnderwriteStarted,
  UnderwritingCompleted{analysis}, DecisionStarted, DecisionMade{decision},
  FairnessEvaluated{fairnessEval}, MemoGenerated{memo}, OfficerReviewRequested,
  OfficerApproved{officerReview}, OfficerDenied{officerReview}, GuardrailRejected{phase, tool,
  reason}, ApplicationFailed{reason}. Commands: submit, startIntake, recordCreditProfile,
  startUnderwrite, recordAnalysis, startDecision, recordDecision, requestOfficerReview,
  recordFairnessEval, recordMemo, officerApprove, officerDeny, recordGuardrailRejection, fail,
  getApplication. emptyState() returns LoanApplicationRecord.initial("") with all Optional
  fields as Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View LoanApplicationView with row type LoanApplicationRow that mirrors
  LoanApplicationRecord exactly (all Optional<T> lifecycle fields preserved). Table updater
  consumes LoanApplicationEntity events. ONE query getAllApplications: SELECT * AS applications
  FROM loan_application_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * LoanEndpoint at /api with POST /loans (body {applicantRef, loanAmount, loanPurpose};
    mints applicationId; calls LoanApplicationEntity.submit(applicantRef, loanAmount,
    loanPurpose); then starts LoanPipelineWorkflow with id "pipeline-" + applicationId;
    returns {applicationId}), GET /loans (list from getAllApplications, sorted newest-first),
    GET /loans/{id} (one row), GET /loans/sse (Server-Sent Events forwarded from the view's
    stream-updates), POST /loans/{id}/review (body {officerId, outcome, notes}; resumes
    LoanPipelineWorkflow), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- LoanTasks.java declaring four Task<R> constants:
    INTAKE_APPLICATION = Task.name("Intake application").description("Gather the business
      credit profile and verify the applicant identity reference").resultConformsTo(
      CreditProfile.class);
    UNDERWRITE_APPLICATION = Task.name("Underwrite application").description("Analyze cash
      flow, value collateral, and identify key risk factors").resultConformsTo(
      UnderwritingAnalysis.class);
    MAKE_DECISION = Task.name("Make decision").description("Compute DSCR, classify risk tier,
      and produce a structured loan recommendation").resultConformsTo(LoanDecision.class);
    GENERATE_REPORT = Task.name("Generate report").description("Compile the underwriting memo
      from the credit profile, analysis, and decision").resultConformsTo(
      UnderwritingMemo.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- LoanPhase.java — enum {INTAKE, UNDERWRITE, DECISION, REPORT}. Each function-tool method is
  annotated with the constant phase, e.g. @FunctionTool(name = "lookupCreditScore",
  phase = LoanPhase.INTAKE).

- IntakeTools.java — @FunctionTool lookupCreditScore(String businessEin) -> CreditScore
  reading from src/main/resources/sample-data/applications/*.json keyed by EIN; @FunctionTool
  fetchBusinessProfile(String businessEin) -> BusinessProfile reading from the same files.

- UnderwriteTools.java — @FunctionTool analyzeCashFlow(Map<String,Object> financials) ->
  CashFlowSummary (deterministic calculation: NOI = revenue - expenses); @FunctionTool
  valuateCollateral(Map<String,Object> collateral) -> CollateralValue (deterministic: LTV =
  loanAmount / estimatedValue).

- DecisionTools.java — @FunctionTool computeDscr(CashFlowSummary cashFlow, BigDecimal
  annualDebtService) -> DscrResult; @FunctionTool classifyRiskTier(CreditScore creditScore,
  DscrResult dscrResult, CollateralValue collateral) -> RiskTier.

- ReportTools.java — @FunctionTool formatDecisionRationale(UnderwritingAnalysis analysis,
  LoanDecision decision) -> RationaleText; @FunctionTool compileConditions(RiskTier riskTier,
  LoanDecision decision) -> List<LoanCondition>.

- PiiSanitizer.java — implements the before-call sanitizer hook. Before each task context is
  assembled, removes ssn, fullLegalName, dateOfBirth, homeAddress from the applicant JSON.
  Logs removed field names at DEBUG (never values). Returns the sanitized map.

- ProtectedAttributeSanitizer.java — implements the before-call sanitizer hook. Masks race,
  ethnicity, sex, age, maritalStatus, nationalOrigin with "[REDACTED]" sentinel before
  UNDERWRITE and DECISION task contexts are assembled.

- PhaseGuardrail.java — implements the before-tool-call hook. Reads the candidate tool call's
  @FunctionTool.phase attribute, looks up the LoanApplicationEntity status by applicationId
  (carried in the TaskDef metadata), applies the accept matrix: INTAKE tools require status ∈
  {SUBMITTED, INTAKE_IN_PROGRESS}; UNDERWRITE tools require status ∈ {INTAKE_COMPLETE,
  UNDERWRITING} AND creditProfile.isPresent(); DECISION tools require status ∈
  {UNDERWRITING_COMPLETE, DECISION_IN_PROGRESS} AND analysis.isPresent(); REPORT tools
  require status ∈ {OFFICER_APPROVED} AND decision.isPresent(). On reject ALSO calls
  LoanApplicationEntity.recordGuardrailRejection(phase, tool, reason).

- FairLendingScorer.java — pure deterministic logic (no LLM). Inputs: LoanDecision,
  UnderwritingAnalysis, CreditProfile. Outputs: FairnessEvalResult with score, flags, and
  rationale. Four checks, one point per check satisfied, starting from a base of 1:
  (1) denial-rate pattern check, (2) rationale completeness (DENY must cite a financial
  criterion), (3) proxy-language absence (PROXY_TERMS = {"neighborhood", "community ties",
  "character", "stability", "lifestyle"}), (4) DSCR correctness (dscrResult.dscr ==
  NOI / annualDebtService ± 0.01). Score range 1-5.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9999 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/applications/*.json — five files (app-001 through app-005),
  each carrying a full seeded applicant record with financials, credit data, and collateral.
  One entry (app-003) includes a protected-attribute field to demonstrate sanitizer coverage.
  One entry (app-004) has a decision rationale in write-decision.json containing a proxy term.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 4 controls (S1, S2, H1, E1) matching the
  mechanisms in Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for finance-analysis domain.

- prompts/loan-underwriting-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Small Business Loan Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of application cards; right = selected-application detail with credit profile,
  underwriting analysis, loan decision, fairness eval chip, officer review panel, rejection-log
  strip). Browser title exactly: <title>Akka Sample: Small Business Loan Agent</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(applicationId)), and
  deserialises into the task's typed return. Mock response files:
    intake-application.json — 5 CreditProfile entries per seeded application, each with a
      tool_calls array containing 1 lookupCreditScore + 1 fetchBusinessProfile call.
    underwrite-application.json — 5 UnderwritingAnalysis entries paired one-to-one,
      tool_calls containing analyzeCashFlow + valuateCollateral.
    make-decision.json — 5 LoanDecision entries paired one-to-one, tool_calls containing
      computeDscr + classifyRiskTier. Includes 1 PHASE-VIOLATING entry whose tool_calls
      starts with formatDecisionRationale (a REPORT-phase tool called during DECISION) —
      selected on the first iteration of every 3rd application (modulo seed) so J2 is
      reproducible. Includes 1 entry whose rationale contains the proxy term "neighborhood
      stability" so J4 is reproducible.
    generate-report.json — 5 UnderwritingMemo entries, tool_calls containing
      formatDecisionRationale + compileConditions.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. LoanUnderwritingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion LoanTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (intakeStep
  60s, underwriteStep 60s, decisionStep 60s, hitlStep no timeout, evalStep 5s, reportStep
  60s, error 5s).
- Lesson 6: every nullable lifecycle field on LoanApplicationRecord is Optional<T>.
- Lesson 7: LoanTasks.java with INTAKE_APPLICATION, UNDERWRITE_APPLICATION, MAKE_DECISION,
  GENERATE_REPORT constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9999 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND themeVariables.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements — no more.
- Single-agent invariant: exactly ONE AutonomousAgent (LoanUnderwritingAgent). FairLendingScorer
  is rule-based and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent; the
  PhaseGuardrail is the runtime mechanism that enforces phase order.
- Task dependency is carried by typed task results: intakeStep writes CreditProfile onto the
  entity, underwriteStep reads it to build its context, decisionStep reads both CreditProfile
  and UnderwritingAnalysis. The agent is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
