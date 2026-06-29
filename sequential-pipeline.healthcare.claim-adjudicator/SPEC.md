# SPEC — claim-adjudication-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Claim Adjudication Agent.
**One-line pitch:** A claims processor submits an insurance claim; one `AdjudicationAgent` walks it through three task phases — **VALIDATE** eligibility and procedure codes, **EVALUATE** coverage against policy terms, **ADJUDICATE** to an approval, denial, or pending-review outcome — with each phase gated on the prior phase's recorded output, PHI sanitised before every tool call, and denials held for human review before finalisation.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a healthcare-payer domain. One `AdjudicationAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the VALIDATE task's typed output becomes the EVALUATE task's instruction context; the EVALUATE task's typed output becomes the ADJUDICATE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` sanitizer guardrail** sits between the agent and every tool call. Before the tool body runs, `PhiSanitizerGuardrail` inspects the call's arguments for PHI fields (member name, date of birth, member ID, provider NPI) and replaces them with masked or tokenised equivalents that preserve clinical meaning without exposing identifiable data downstream. The sanitiser fires on every tool call, regardless of phase. A `PhiRedacted` audit event is emitted for each field masked, giving the compliance team a full redaction log without requiring tool authors to implement their own masking.
- A **human-in-the-loop hold** is triggered by `ClaimAdjudicationWorkflow` whenever `AdjudicationDecision.outcome == DENIED`. The workflow pauses at `humanReviewStep`, sets the claim to `PENDING_REVIEW`, and waits for an explicit `approveDenial` or `overrideToApprove` command from a human reviewer before the claim transitions to `DENIED` or `APPROVED`. The hold is bounded by a configurable review timeout; an unreviewed denial after the timeout transitions to `ESCALATED`.
- An **`on-decision-eval`** runs immediately after `DecisionReached` lands, as `evalStep` inside the workflow. A deterministic, rule-based `AdjudicationScorer` (no LLM call — the eval is rule-based on purpose) checks that every adjudication decision cites at least two policy rules from the evaluated coverage, that the cited procedure codes appear in the validated claim, that the decision outcome is consistent with the coverage evaluation result, and that the evidence completeness score (number of policy rules cited ÷ expected rules for the procedure family) meets a minimum threshold.

## 3. User-facing flows

The user opens the App UI tab.

1. The claims processor types or uploads a **claim** (or picks one of three seeded claims — a routine lab order `LAB-001`, an outpatient surgery `SURG-002`, and a mental health visit `MH-003`).
2. The processor clicks **Submit claim**. The UI POSTs to `/api/claims` and receives a `claimId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `VALIDATING` — the workflow has started `validateStep` and the agent has been handed the VALIDATE task.
4. Within ~10–20 s the card reaches `VALIDATED`. The card detail shows the validation result: eligibility confirmed or flagged, procedure codes verified, any pre-authorisation requirements noted.
5. Within ~10–20 s more the card reaches `EVALUATED`. The coverage evaluation is visible — which policy rules apply, what the covered amount is, and any exclusion findings.
6. Within ~10–20 s more the card reaches one of two branches:
   - **Approval path**: status becomes `DECIDED → EVALUATED` with outcome `APPROVED`. The right pane shows the full `AdjudicationDecision` — policy rules cited, covered amount, explanation — plus an eval score chip (1–5).
   - **Denial path**: status becomes `PENDING_REVIEW`. A reviewer receives a notification and visits the claim detail to read the denial reason and coverage evaluation. Clicking **Approve denial** transitions the claim to `DENIED`; clicking **Override to approve** transitions it to `APPROVED`.
7. After the review action (or for the direct-approval path), the claim reaches `EVALUATED` with its final decision and eval score visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ClaimEndpoint` | `HttpEndpoint` | `/api/claims/*` — submit, list, get, SSE, review action; serves `/api/metadata/*`. | — | `ClaimEntity`, `ClaimView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ClaimEntity` | `EventSourcedEntity` | Per-claim lifecycle: created → validating → validated → evaluating → evaluated → deciding → decided → pending-review → approved/denied → evaluated. Source of truth. | `ClaimEndpoint`, `ClaimAdjudicationWorkflow` | `ClaimView` |
| `ClaimAdjudicationWorkflow` | `Workflow` | One workflow per claim. Steps: `validateStep` → `evaluateStep` → `adjudicateStep` → `humanReviewStep` (conditional on denial) → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `ClaimEndpoint` after `CREATED` | `AdjudicationAgent`, `ClaimEntity` |
| `AdjudicationAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `AdjudicationTasks.java`: `VALIDATE_CLAIM` → `ValidationResult`, `EVALUATE_COVERAGE` → `CoverageEvaluation`, `ADJUDICATE_CLAIM` → `AdjudicationDecision`. Each task is registered with the phase-appropriate function tools. | invoked by `ClaimAdjudicationWorkflow` | returns typed results |
| `ValidateTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `checkEligibility(memberId)` and `verifyProcedureCodes(codes)`. Reads from `src/main/resources/sample-data/members/*.json` and `procedure-codes/*.json` for deterministic offline output. | called from VALIDATE task | returns `EligibilityResult` / `List<CodeVerification>` |
| `EvaluateTools` | function-tools class | Implements `lookupPolicyRules(procedureFamily)` and `computeCoveredAmount(validation, rules)`. Pure in-memory policy-rule lookups against `src/main/resources/sample-data/policy-rules/*.json`. | called from EVALUATE task | returns `List<PolicyRule>` / `CoverageAmount` |
| `AdjudicateTools` | function-tools class | Implements `buildDecisionRationale(evaluation, claim)` and `classifyOutcome(evaluation)`. Pure deterministic logic. | called from ADJUDICATE task | returns `DecisionRationale` / `ClaimOutcome` |
| `PhiSanitizerGuardrail` | `before-tool-call` guardrail (registered on `AdjudicationAgent`) | Inspects every tool call's arguments for PHI fields (member name, date of birth, member ID, provider NPI). Replaces PHI with masked equivalents before the tool body executes. Emits `PhiRedacted` audit event per masked field. Never rejects — always passes a sanitised call through. | every tool call on every task | sanitised call / `PhiRedacted` event |
| `AdjudicationScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `AdjudicationDecision`, `CoverageEvaluation`, `ValidationResult`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `ClaimView` | `View` | Read model: one row per claim for the UI. | `ClaimEntity` events | `ClaimEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MemberInfo(String tokenisedMemberId, String planId, String groupId) {}

record ProcedureCode(String code, String system, String description) {}

record EligibilityResult(
    boolean eligible,
    String planName,
    String coverageType,
    List<String> priorAuthRequired,
    Instant checkedAt
) {}

record CodeVerification(String code, boolean recognised, String procedureFamily) {}

record ValidationResult(
    EligibilityResult eligibility,
    List<CodeVerification> codeVerifications,
    List<String> validationWarnings,
    Instant validatedAt
) {}

record PolicyRule(String ruleId, String description, String applicability, boolean exclusion) {}

record CoverageAmount(
    String currency,
    int coveredUnits,
    double coveredPercent,
    double memberResponsibility
) {}

record CoverageEvaluation(
    List<PolicyRule> applicableRules,
    List<PolicyRule> exclusionRules,
    CoverageAmount coverageAmount,
    String coverageSummary,
    Instant evaluatedAt
) {}

record DecisionRationale(String narrative, List<String> citedRuleIds) {}

enum ClaimOutcome { APPROVED, DENIED, PENDING_REVIEW }

record AdjudicationDecision(
    ClaimOutcome outcome,
    DecisionRationale rationale,
    CoverageAmount approvedAmount,
    String denialCode,       // null when outcome != DENIED
    Instant decidedAt
) {}

record EvalResult(
    int score,             // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record PhiRedaction(String field, String maskedValue, Instant redactedAt) {}

record ClaimRecord(
    String claimId,
    MemberInfo memberInfo,
    List<ProcedureCode> procedureCodes,
    String serviceDate,
    String providerId,
    Optional<ValidationResult> validation,
    Optional<CoverageEvaluation> coverage,
    Optional<AdjudicationDecision> decision,
    Optional<EvalResult> eval,
    ClaimStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<PhiRedaction> phiRedactions,
    List<GuardrailRejection> guardrailRejections
) {}

enum ClaimStatus {
    CREATED, VALIDATING, VALIDATED, EVALUATING, EVALUATED_COVERAGE,
    DECIDING, DECIDED, PENDING_REVIEW, APPROVED, DENIED, ESCALATED,
    ADJUDICATION_EVALUATED, FAILED
}
```

Events on `ClaimEntity`: `ClaimCreated`, `ValidationStarted`, `ClaimValidated`, `EvaluationStarted`, `CoverageEvaluated`, `AdjudicationStarted`, `DecisionReached`, `HumanReviewRequested`, `DenialApproved`, `DenialOverridden`, `ClaimEscalated`, `EvaluationScored`, `PhiRedacted`, `GuardrailRejected`, `ClaimFailed`.

Every nullable lifecycle field on the `ClaimRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/claims` — body `{ memberInfo, procedureCodes, serviceDate, providerId }` → `{ claimId }`.
- `GET /api/claims` — list all claims, newest-first.
- `GET /api/claims/{id}` — one claim.
- `GET /api/claims/sse` — Server-Sent Events; one event per state transition.
- `POST /api/claims/{id}/review` — body `{ action: "APPROVE_DENIAL" | "OVERRIDE_TO_APPROVE", reviewerId, notes }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Claim Adjudication Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of claims (status pill + claim type + age + outcome badge when decided) and a right pane with the selected claim's detail — member info (tokenised), procedure codes, validation result, coverage evaluation, adjudication decision, eval score chip, PHI-redaction log strip, and a reviewer action panel for claims in `PENDING_REVIEW`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — `before-tool-call` PHI sanitizer**: `PhiSanitizerGuardrail` is registered on `AdjudicationAgent` and runs before every tool call. It inspects the call's arguments for PHI fields defined in the `PhiFieldRegistry` (member name → `name`, date of birth → `dob`, member ID → `memberId`, provider NPI → `npi`). Each matched field is replaced with a masked token of the form `[REDACTED:<field>:<sha1(value)[0:8]>]` — length-preserving enough to avoid breaking downstream parsing, deterministic enough to let downstream deduplication work. A `PhiRedacted{field, maskedValue}` event is emitted per masked field so the compliance team has a full, queryable redaction log. The guardrail never rejects — it always passes a sanitised version of the call through. Failure to sanitise (e.g., unrecognised PHI schema version) transitions the claim to `FAILED` with reason `phi-sanitization-error`.
- **H1 — human-in-the-loop denial review**: `ClaimAdjudicationWorkflow.humanReviewStep` is triggered whenever `AdjudicationDecision.outcome == DENIED`. The workflow emits `HumanReviewRequested{decisionSummary, denialCode, reviewDeadline}` on the entity and blocks at the step, waiting for either `approveDenial` or `overrideToApprove` command from `ClaimEndpoint`. The step has a `stepTimeout` of 72 h in production configuration (10 s in dev mode, controlled by `akka.javasdk.dev-mode.hitl-timeout`). If the timeout expires, `ClaimEscalated` is emitted and the claim status becomes `ESCALATED`. On `DenialApproved`, the claim transitions to `DENIED`; on `DenialOverridden`, it transitions to `APPROVED`.
- **E1 — `on-decision-eval`**: runs immediately after `DecisionReached` lands, as `evalStep` inside the workflow. `AdjudicationScorer` is deterministic and rule-based (no LLM call). Four checks, one point per check satisfied, on a base of 1: (1) rule citation count — `rationale.citedRuleIds.size() >= 2`; (2) code provenance — every `citedRuleId` maps to a rule whose `applicability` matches a recognised procedure family from the validated codes; (3) outcome consistency — if `evaluation.exclusionRules` is non-empty and `evaluation.coverageAmount.coveredPercent < 0.01`, outcome must be `DENIED` or `PENDING_REVIEW`; (4) evidence completeness — `citedRuleIds.size() >= expectedRuleCount(procedureFamily)`. Emits `EvaluationScored{score: 1-5, rationale}`. Scores ≤ 2 flag the claim for secondary review.

## 9. Agent prompts

- `AdjudicationAgent` → `prompts/adjudication-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. The prompt also instructs the agent that PHI sanitisation is handled by the runtime — it should not attempt to mask, anonymise, or inspect PHI fields itself.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Processor submits seeded claim `LAB-001`; within 90 s the claim reaches `ADJUDICATION_EVALUATED` with validation present, ≥ 2 policy rules cited, and an eval score chip showing 4 or 5.
2. **J2** — The agent's first iteration on a claim calls an ADJUDICATE-phase tool (`buildDecisionRationale`) before `CoverageEvaluated` has been recorded (mock LLM path). `PhiSanitizerGuardrail` still fires (sanitiser always runs); a separate phase-violation check in the guardrail rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the claim eventually reaches `ADJUDICATION_EVALUATED`.
3. **J3** — A claim routed to `DENIED` (seeded claim `DENIED-004`) triggers `humanReviewStep`. The UI shows the claim in `PENDING_REVIEW` with a reviewer action panel. After the reviewer clicks **Approve denial**, the claim transitions to `DENIED → ADJUDICATION_EVALUATED`.
4. **J4** — A claim adjudicated with only one cited policy rule (mock path) scores 2; the rationale names "rule citation count below minimum (1 < 2)"; the card border highlights red.
5. **J5** — PHI-redaction log strip on every claim card shows at least two `PhiRedacted` entries (`memberId`, `npi`). No raw PHI values appear anywhere in the UI or API responses.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named claim-adjudication-agent demonstrating the sequential-pipeline x
healthcare cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-healthcare-claim-adjudicator.
Java package io.akka.samples.claimadjudicationagent. Akka 3.6.0. HTTP port 9530.

Components to wire (exactly):

- 1 AutonomousAgent AdjudicationAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/adjudication-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the VALIDATE, EVALUATE, and ADJUDICATE tool sets are ALL registered
  on the agent; phase gating is secondary (sanitiser is primary) and both are the job of
  PhiSanitizerGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (PhiSanitizerGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On phase-violation rejection the agent loop retries within
  its 4-iteration budget.

- 1 Workflow ClaimAdjudicationWorkflow per claimId with five steps:
  * validateStep — emits ValidationStarted on the entity, then calls componentClient
    .forAutonomousAgent(AdjudicationAgent.class, "agent-" + claimId).runSingleTask(
      TaskDef.instructions("ClaimId: " + claimId + "\nPhase: VALIDATE\nMember plan: "
        + planId + "\nProcedure codes: " + codes + "\nUse checkEligibility and
        verifyProcedureCodes to validate the claim.")
        .metadata("claimId", claimId)
        .metadata("phase", "VALIDATE")
        .taskType(AdjudicationTasks.VALIDATE_CLAIM)
    ). Reads forTask(taskId).result(VALIDATE_CLAIM) to get ValidationResult. Writes
    ClaimEntity.recordValidation(validationResult). WorkflowSettings.stepTimeout 90s.
  * evaluateStep — emits EvaluationStarted, then runSingleTask with TaskDef.instructions
    (formatEvaluateContext(validationResult, claimId)) and metadata.phase = "EVALUATE",
    taskType EVALUATE_COVERAGE. Writes ClaimEntity.recordCoverage(coverageEvaluation).
    stepTimeout 90s.
  * adjudicateStep — emits AdjudicationStarted, then runSingleTask with TaskDef.instructions
    (formatAdjudicateContext(coverage, validationResult, claimId)) and metadata.phase =
    "ADJUDICATE", taskType ADJUDICATE_CLAIM. Writes ClaimEntity.recordDecision(decision).
    stepTimeout 90s. After recordDecision, if decision.outcome == DENIED, workflow transitions
    to humanReviewStep; otherwise transitions directly to evalStep.
  * humanReviewStep — emits HumanReviewRequested{decisionSummary, denialCode, reviewDeadline}
    on the entity. Blocks waiting for ClaimEndpoint to post approveDenial or overrideToApprove.
    stepTimeout 72h (dev-mode override: 10s via akka.javasdk.dev-mode.hitl-timeout).
    On timeout emits ClaimEscalated and ends. On approveDenial emits DenialApproved and
    advances to evalStep. On overrideToApprove emits DenialOverridden and advances to evalStep.
  * evalStep — runs the deterministic AdjudicationScorer over (decision, coverage, validation)
    and writes ClaimEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ClaimAdjudicationWorkflow::error). The error step writes
  ClaimFailed and ends.

- 1 EventSourcedEntity ClaimEntity (one per claimId). State ClaimRecord{claimId,
  memberInfo: MemberInfo, procedureCodes: List<ProcedureCode>, serviceDate: String,
  providerId: String, validation: Optional<ValidationResult>,
  coverage: Optional<CoverageEvaluation>, decision: Optional<AdjudicationDecision>,
  eval: Optional<EvalResult>, status: ClaimStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, phiRedactions: List<PhiRedaction>,
  guardrailRejections: List<GuardrailRejection>}.
  ClaimStatus enum: CREATED, VALIDATING, VALIDATED, EVALUATING, EVALUATED_COVERAGE,
  DECIDING, DECIDED, PENDING_REVIEW, APPROVED, DENIED, ESCALATED,
  ADJUDICATION_EVALUATED, FAILED.
  Events: ClaimCreated, ValidationStarted, ClaimValidated, EvaluationStarted,
  CoverageEvaluated, AdjudicationStarted, DecisionReached, HumanReviewRequested,
  DenialApproved, DenialOverridden, ClaimEscalated, EvaluationScored, PhiRedacted,
  GuardrailRejected, ClaimFailed.
  Commands: create, startValidation, recordValidation, startEvaluation, recordCoverage,
  startAdjudication, recordDecision, requestHumanReview, approveDenial, overrideToApprove,
  escalate, recordEvaluation, recordPhiRedaction, recordGuardrailRejection, fail,
  getClaim. emptyState() returns ClaimRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View ClaimView with row type ClaimRow that mirrors ClaimRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes ClaimEntity events. ONE query
  getAllClaims: SELECT * AS claims FROM claim_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ClaimEndpoint at /api with POST /claims (body {memberInfo, procedureCodes, serviceDate,
    providerId}; mints claimId; calls ClaimEntity.create(...); then starts
    ClaimAdjudicationWorkflow with id "adj-" + claimId; returns {claimId}), GET /claims
    (list from getAllClaims, sorted newest-first), GET /claims/{id} (one row), GET
    /claims/sse (Server-Sent Events forwarded from the view's stream-updates), POST
    /claims/{id}/review (body {action, reviewerId, notes}; dispatches approveDenial or
    overrideToApprove command to ClaimEntity; returns 204), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- AdjudicationTasks.java declaring three Task<R> constants:
    VALIDATE_CLAIM = Task.name("Validate claim").description("Check member eligibility and
      verify procedure codes against the payer's recognised code system")
      .resultConformsTo(ValidationResult.class);
    EVALUATE_COVERAGE = Task.name("Evaluate coverage").description("Look up applicable policy
      rules and compute the covered amount for the validated claim")
      .resultConformsTo(CoverageEvaluation.class);
    ADJUDICATE_CLAIM = Task.name("Adjudicate claim").description("Produce a decision with
      outcome (APPROVED/DENIED/PENDING_REVIEW), rationale citing policy rules, and approved
      amount").resultConformsTo(AdjudicationDecision.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- AdjudicationPhase.java — enum {VALIDATE, EVALUATE, ADJUDICATE}. Each function-tool method
  carries the constant phase.

- ValidateTools.java — @FunctionTool checkEligibility(String tokenisedMemberId, String planId)
  -> EligibilityResult reading from src/main/resources/sample-data/members/*.json;
  @FunctionTool verifyProcedureCodes(List<ProcedureCode> codes) -> List<CodeVerification>
  reading from src/main/resources/sample-data/procedure-codes/*.json.

- EvaluateTools.java — @FunctionTool lookupPolicyRules(String procedureFamily)
  -> List<PolicyRule> reading from src/main/resources/sample-data/policy-rules/*.json;
  @FunctionTool computeCoveredAmount(ValidationResult validation, List<PolicyRule> rules)
  -> CoverageAmount (deterministic calculation: sum applicable rules, subtract exclusions,
  apply plan deductible from EligibilityResult).

- AdjudicateTools.java — @FunctionTool buildDecisionRationale(CoverageEvaluation evaluation,
  String claimId) -> DecisionRationale (narrative from coverage summary + cited rule ids);
  @FunctionTool classifyOutcome(CoverageEvaluation evaluation) -> ClaimOutcome (deterministic:
  DENIED if exclusionRules non-empty AND coveredPercent < 0.01, APPROVED if coveredPercent
  >= 0.80, else PENDING_REVIEW).

- PhiSanitizerGuardrail.java — implements the before-tool-call hook. Inspects every argument
  for fields matching PhiFieldRegistry (name, dob, memberId, npi). Replaces each matched
  field value with "[REDACTED:<field>:<sha1(value)[0:8]>]". ALSO checks the candidate
  tool call's phase against AdjudicationPhase, current ClaimEntity status, and the accept
  matrix: VALIDATE tools require status ∈ {CREATED, VALIDATING}; EVALUATE tools require
  status ∈ {VALIDATED, EVALUATING} AND validation.isPresent(); ADJUDICATE tools require
  status ∈ {EVALUATED_COVERAGE, DECIDING} AND coverage.isPresent(). On phase mismatch,
  returns Guardrail.reject("phase-violation: <tool> requires <precondition>, saw <status>")
  AND calls ClaimEntity.recordGuardrailRejection. On PHI field redaction, emits
  ClaimEntity.recordPhiRedaction(field, maskedValue) and PASSES the sanitised call through.

- AdjudicationScorer.java — pure deterministic logic (no LLM). Inputs: AdjudicationDecision,
  CoverageEvaluation, ValidationResult. Outputs: EvalResult with score and rationale.
  Four checks, one point per check satisfied, starting from base 1:
  (1) rule citation count: rationale.citedRuleIds.size() >= 2.
  (2) code provenance: every citedRuleId maps to a rule whose applicability covers at least
  one recognised procedure family from validationResult.codeVerifications.
  (3) outcome consistency: if exclusionRules non-empty and coveredPercent < 0.01, outcome
  must be DENIED or PENDING_REVIEW.
  (4) evidence completeness: citedRuleIds.size() >= expectedRuleCount(primaryProcedureFamily).
  Score range 1-5. Rationale names the first failing check.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9530,
  akka.javasdk.dev-mode.hitl-timeout = 10s, and the three model-provider blocks (anthropic
  claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash) reading the canonical
  env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/claims.jsonl with 6 seeded claim lines covering LAB-001,
  SURG-002, MH-003, DENIED-004, ESCALATED-005, and one custom-procedure claim.

- src/main/resources/sample-data/members/*.json — three member files (plan PPO-100,
  HMO-200, HDHP-300) each with eligibility status, deductible, and out-of-pocket max.

- src/main/resources/sample-data/procedure-codes/*.json — procedure code files for lab,
  surgery, and mental-health families, each with description and recognised flag.

- src/main/resources/sample-data/policy-rules/*.json — policy rule files for each procedure
  family, each with ruleId, description, applicability, and exclusion flag.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, H1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.phi = true (claims contain
  PHI), decisions.authority_level = binding (an adjudication decision is binding on the
  payer and member), oversight.human_in_loop = true (denials require human review),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline.

- prompts/adjudication-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Claim Adjudication Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of claim cards; right = selected-claim detail with member info (tokenised),
  procedure codes, validation result, coverage evaluation, adjudication decision, eval-score
  chip, PHI-redaction log strip, and reviewer action panel for PENDING_REVIEW claims).
  Browser title exactly: <title>Akka Sample: Claim Adjudication Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(claimId)), and
  deserialises into the task's typed return. Each entry carries a "tool_calls" array the
  mock replays in order before returning the final typed result.
- Per-task mock-response shapes:
    validate-claim.json — 6 ValidationResult entries for the seeded claims. One entry per
      seeded claim, plus one deliberately PHASE-VIOLATING entry whose tool_calls array starts
      with buildDecisionRationale (an ADJUDICATE-phase tool called during VALIDATE) — the
      guardrail rejects it (after sanitising PHI), the mock falls through to a normal
      validate sequence. Selected on the first iteration of every 3rd claim (modulo seed)
      so J2 is reproducible.
    evaluate-coverage.json — 6 CoverageEvaluation entries. The DENIED-004 entry returns
      exclusionRules non-empty and coveredPercent < 0.01 to trigger humanReviewStep.
    adjudicate-claim.json — 6 AdjudicationDecision entries. One entry cites only 1 policy
      rule (below the minimum of 2) — scores 2 in J4. The DENIED-004 entry carries
      outcome = DENIED and a denial code.
- A MockModelProvider.seedFor(claimId) helper makes per-claim selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AdjudicationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AdjudicationTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (validateStep
  90s, evaluateStep 90s, adjudicateStep 90s, humanReviewStep 72h / dev 10s, evalStep 5s,
  error 5s).
- Lesson 6: every nullable lifecycle field on the ClaimRecord row record is Optional<T>.
- Lesson 7: AdjudicationTasks.java with VALIDATE_CLAIM, EVALUATE_COVERAGE, ADJUDICATE_CLAIM
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9530 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: index.html includes mermaid CSS overrides AND themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (AdjudicationAgent). The
  on-decision eval is rule-based (AdjudicationScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  PhiSanitizerGuardrail is the runtime mechanism that enforces both PHI sanitisation and
  phase order. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: validateStep writes ValidationResult
  onto the entity, evaluateStep reads it and builds the EVALUATE task's instruction context,
  adjudicateStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
