# SPEC — medical-preauth

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Medical Pre-Authorization.
**One-line pitch:** A clinician submits a prior-authorization request; one `PreAuthAgent` walks it through three task phases — **VALIDATE** the clinical submission, **REVIEW** matching payer policies, **DETERMINE** an approval or denial — with PHI stripped before every LLM call, denials held for human acknowledgement, and every determination scored for clinical completeness.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a healthcare payer domain. One `PreAuthAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the VALIDATE task's typed output becomes the REVIEW task's instruction context; the REVIEW task's typed output becomes the DETERMINE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail backed by a PHI sanitizer** sits between the agent and every tool call. `PhiSanitizer` strips or hashes regulated patient identifiers (MRN, date of birth, full name, SSN) from every tool call payload before the call reaches the LLM. Every field removed is recorded in a `SanitizationApplied` event on `PreAuthEntity` for audit. The phase portion of the same guardrail also checks that each tool is called only in its declared phase — a DETERMINE-phase tool called while the entity has not yet recorded `PolicyReviewed` is rejected before the tool body runs.
- A **human-in-the-loop hold (`DenialHoldStep`)** runs immediately after any determination whose outcome is `DENIED`. The workflow pauses and waits up to 72 hours for a human reviewer to click **Acknowledge** via `POST /api/auth-requests/{id}/acknowledge`. If no acknowledgement arrives within 72 hours, the workflow escalates the request and writes `EscalationRequired` onto the entity.
- An **`on-decision-eval`** runs immediately after `DeterminationWritten` lands, as `evalStep` inside the workflow. A deterministic, rule-based `ClinicalCompletenessScorer` (no LLM call) checks that every cited procedure code appears in the validated request, that at least one policy article is cited per determination, that the outcome is one of the valid enum values, and that the rationale word count exceeds the minimum threshold (≥ 40 words for denials, ≥ 15 words for approvals).

The blueprint shows that a sequential pipeline in a regulated domain is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce PHI isolation, per-phase tool access, and the clinical dependency contract.

## 3. User-facing flows

The user opens the App UI tab.

1. The clinician selects a **request type** (Prior Authorization) and enters the **member ID**, **procedure code**, **diagnosis code (ICD-10)**, and **treating physician NPI** into the submission form, or picks one of three seeded requests (`DEXA bone density scan`, `Knee arthroscopy`, `Continuous glucose monitor`).
2. The clinician clicks **Submit request**. The UI POSTs to `/api/auth-requests` and receives a `requestId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `VALIDATING` — the workflow has started `validateStep` and the agent has been handed the VALIDATE task.
4. Within ~15–25 s the card reaches `VALIDATED`. The right pane shows the `ValidationResult` — member eligibility confirmed, procedure code recognized, ICD-10 linkage check passed. The agent's VALIDATE task returned; the workflow recorded `RequestValidated` and ran the REVIEW task.
5. Within ~15–25 s more the card reaches `POLICY_REVIEWED`. The `PolicyReview` is visible (matched policy articles, criteria met/unmet list, evidence citations).
6. Within ~15–25 s more the card reaches `DETERMINED`. If the outcome is `APPROVED`, the card moves immediately to `EVALUATED`. If the outcome is `DENIED`, the card transitions to `PENDING_HUMAN_REVIEW` — the DenialHoldStep is active and a reviewer must acknowledge before the determination finalizes.
7. Once `EVALUATED`, the right pane shows the full typed `Determination` — outcome, rationale, policy citations, procedure coverage notes — plus a clinical completeness score chip (1–5) and a one-line eval rationale.
8. A human reviewer can open any `PENDING_HUMAN_REVIEW` request and click **Acknowledge denial** to advance it to `EVALUATED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PreAuthEndpoint` | `HttpEndpoint` | `/api/auth-requests/*` — submit, list, get, SSE, acknowledge; serves `/api/metadata/*`. | — | `PreAuthEntity`, `PreAuthView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PreAuthEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → validating → validated → reviewing → policy-reviewed → determining → determined → (pending-human-review) → evaluated. Source of truth. | `PreAuthEndpoint`, `PreAuthWorkflow` | `PreAuthView` |
| `PreAuthWorkflow` | `Workflow` | One workflow per requestId. Steps: `validateStep` → `reviewStep` → `determineStep` → `evalStep` → `humanHoldStep` (conditional on DENIED outcome). Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `PreAuthEndpoint` after `SUBMITTED` | `PreAuthAgent`, `PreAuthEntity` |
| `PreAuthAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `PreAuthTasks.java`: `VALIDATE_REQUEST` → `ValidationResult`, `REVIEW_POLICY` → `PolicyReview`, `DETERMINE_OUTCOME` → `Determination`. Each task is registered with the phase-appropriate function tools. | invoked by `PreAuthWorkflow` | returns typed results |
| `ValidateTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `checkMemberEligibility(memberId)` and `validateProcedureCode(procedureCode, icd10Code)`. Reads from `src/main/resources/sample-data/members/*.json` and `procedure-codes/*.json`. | called from VALIDATE task | returns `EligibilityCheck` / `ProcedureCheck` |
| `ReviewTools` | function-tools class | Implements `matchPolicyArticles(procedureCode, diagnosisCode)` and `evaluateCriteria(policyArticles, clinicalNotes)`. Reads from `src/main/resources/sample-data/policies/*.json`. | called from REVIEW task | returns `List<PolicyArticle>` / `CriteriaEvaluation` |
| `DetermineTools` | function-tools class | Implements `buildRationale(criteriaEvaluation, policyArticles)` and `classifyOutcome(criteriaEvaluation)`. Pure in-memory transformations on the REVIEW phase's typed output. | called from DETERMINE task | returns `String` / `AuthOutcome` |
| `PhiSanitizer` | `before-tool-call` guardrail (registered on `PreAuthAgent`) | Strips or hashes PHI fields (MRN, DOB, full name, SSN) from every outbound tool call payload. Records a `SanitizationApplied` audit event per field removed. Also enforces phase-gate: rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | sanitize + accept / structured-reject |
| `ClinicalCompletenessScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `Determination`, `PolicyReview`, `ValidationResult`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `PreAuthView` | `View` | Read model: one row per request for the UI. | `PreAuthEntity` events | `PreAuthEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MemberInfo(
    String memberId,          // sanitized to hash before LLM
    String planId,
    String groupNumber,
    boolean activeEligibility
) {}

record ProcedureRequest(
    String procedureCode,     // CPT code
    String diagnosisCode,     // ICD-10 code
    String treatingPhysicianNpi,
    String requestingFacility,
    String clinicalNotes      // free text; PHI-sanitized before LLM
) {}

record EligibilityCheck(
    boolean eligible,
    String planType,
    String coverageTier,
    Instant verifiedAt
) {}

record ProcedureCheck(
    boolean codeRecognized,
    String codeDescription,
    boolean icd10Linked,
    Instant checkedAt
) {}

record ValidationResult(
    EligibilityCheck eligibility,
    ProcedureCheck procedureCheck,
    boolean validationPassed,
    Instant validatedAt
) {}

record PolicyArticle(
    String articleId,
    String title,
    String criteriaText,
    String policyVersion
) {}

record CriterionResult(
    String criterionId,
    String description,
    boolean met,
    String evidence
) {}

record CriteriaEvaluation(
    List<CriterionResult> criteria,
    int metCount,
    int totalCount,
    Instant evaluatedAt
) {}

record PolicyReview(
    List<PolicyArticle> matchedArticles,
    CriteriaEvaluation criteriaEvaluation,
    Instant reviewedAt
) {}

enum AuthOutcome { APPROVED, DENIED, PENDING_ADDITIONAL_INFO }

record Determination(
    AuthOutcome outcome,
    String rationale,
    List<String> citedArticleIds,
    List<String> coveredProcedureCodes,
    Instant determinedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record SanitizationRecord(
    String field,
    String hashedToken,
    String tool,
    Instant sanitizedAt
) {}

record PreAuthRecord(
    String requestId,
    Optional<MemberInfo> memberInfo,
    Optional<ProcedureRequest> procedureRequest,
    Optional<ValidationResult> validationResult,
    Optional<PolicyReview> policyReview,
    Optional<Determination> determination,
    Optional<EvalResult> eval,
    PreAuthStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<SanitizationRecord> sanitizationLog,
    List<PhaseRejection> phaseRejections
) {}

enum PreAuthStatus {
    SUBMITTED, VALIDATING, VALIDATED, REVIEWING, POLICY_REVIEWED,
    DETERMINING, DETERMINED, PENDING_HUMAN_REVIEW,
    EVALUATED, ESCALATED, FAILED
}
```

Events on `PreAuthEntity`: `RequestSubmitted`, `ValidationStarted`, `RequestValidated`, `ReviewStarted`, `PolicyReviewed`, `DetermineStarted`, `DeterminationWritten`, `EvaluationScored`, `HumanApprovalRequested`, `HumanApprovalReceived`, `EscalationRequired`, `SanitizationApplied`, `PhaseGuardrailRejected`, `RequestFailed`.

Every nullable lifecycle field on the `PreAuthRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/auth-requests` — body `{ memberId, procedureCode, diagnosisCode, treatingPhysicianNpi, clinicalNotes }` → `{ requestId }`.
- `GET /api/auth-requests` — list all requests, newest-first.
- `GET /api/auth-requests/{id}` — one request.
- `GET /api/auth-requests/sse` — Server-Sent Events; one event per state transition.
- `POST /api/auth-requests/{id}/acknowledge` — body `{ reviewerNotes }` → `{ status }`. Advances a `PENDING_HUMAN_REVIEW` request to `EVALUATED`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Medical Pre-Authorization</title>`.

The App UI tab is a two-column layout: a left rail with the live list of authorization requests (status pill + procedure code + age + red dot if any PHI sanitization or phase rejection fired) and a right pane with the selected request's detail — member eligibility summary, procedure validation result, policy review (matched articles, criteria met/unmet), determination outcome chip, eval score chip, and a sanitization-log strip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PHI sanitizer (`before-tool-call`)**: `PhiSanitizer` is registered on `PreAuthAgent` and runs before every tool call. Before passing the call's payload to the LLM, it scans for a defined PHI field list (`memberId`, `dateOfBirth`, `patientName`, `ssn`, and any string matching a 9-digit SSN pattern or a `YYYY-MM-DD` date adjacent to a name field). Each detected field is replaced with `sha256(field + requestId).substring(0, 12)` — a stable, request-scoped token that preserves cross-call linkage without exposing the identifier. Every replacement emits `SanitizationApplied{field, hashedToken, tool, sanitizedAt}` on the entity. If the sanitizer encounters a payload it cannot parse (malformed JSON or a binary attachment), it blocks the call entirely and emits `PhaseGuardrailRejected` with reason `sanitization-parse-failure`. The phase-gate portion of the same hook enforces the per-phase accept matrix: `VALIDATE` tools require `status ∈ {SUBMITTED, VALIDATING}`; `REVIEW` tools require `status ∈ {VALIDATED, REVIEWING}` AND `validationResult.isPresent()` AND `validationResult.get().validationPassed()`; `DETERMINE` tools require `status ∈ {POLICY_REVIEWED, DETERMINING}` AND `policyReview.isPresent()`.
- **H1 — human-in-the-loop hold (denial gate)**: `humanHoldStep` runs after `evalStep` when the determination outcome is `DENIED`. It emits `HumanApprovalRequested` on the entity, transitions status to `PENDING_HUMAN_REVIEW`, and waits for a `POST /api/auth-requests/{id}/acknowledge` call. The step's `stepTimeout` is 72 hours. On timeout, `PreAuthWorkflow` writes `EscalationRequired` and transitions to `ESCALATED`. On acknowledgement, `HumanApprovalReceived` is written and status transitions to `EVALUATED`. `APPROVED` and `PENDING_ADDITIONAL_INFO` determinations bypass `humanHoldStep` entirely and go directly to `EVALUATED`.
- **E1 — `on-decision-eval`**: runs immediately after `DeterminationWritten` lands, as `evalStep` inside the workflow. `ClinicalCompletenessScorer` is a deterministic rule-based scorer (no LLM call). Four checks, one point per check satisfied, on a base of 1: (1) **procedure code coverage** — every `Determination.coveredProcedureCodes[i]` appears in `ValidationResult.procedureCheck.procedureCode`; (2) **policy citation** — `Determination.citedArticleIds` is non-empty and every cited id appears in `PolicyReview.matchedArticles[].articleId`; (3) **rationale adequacy** — `rationale.split("\\s+").length >= (outcome == DENIED ? 40 : 15)`; (4) **outcome validity** — `outcome ∈ {APPROVED, DENIED, PENDING_ADDITIONAL_INFO}`. Emits `EvaluationScored{score:1..5, rationale}`.

## 9. Agent prompts

- `PreAuthAgent` → `prompts/pre-auth-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Clinician submits the seeded request `DEXA bone density scan`; within 90 s the request reaches `EVALUATED` with a non-empty `ValidationResult`, ≥ 1 matched policy article, a `Determination` with outcome `APPROVED`, and an eval score chip showing 5/5.
2. **J2** — The PHI sanitizer fires on any outbound tool call payload that includes the raw `memberId`. The `SanitizationApplied` event is visible in the sanitization-log strip; the LLM receives the hashed token, not the original MRN.
3. **J3** — A `DENIED` determination triggers `DenialHoldStep`; the request sits in `PENDING_HUMAN_REVIEW`. A reviewer clicks **Acknowledge denial**; the workflow resumes; the determination finalizes at `EVALUATED`. The timeline strip shows the hold duration.
4. **J4** — A determination that cites a policy article id absent from `PolicyReview.matchedArticles` is scored 1 by `ClinicalCompletenessScorer`. The UI flags the card and the right-pane eval section shows the rationale naming the missing citation.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named medical-preauth demonstrating the sequential-pipeline x healthcare cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-healthcare-medical-preauth. Java package
io.akka.samples.medicalpreauthorization. Akka 3.6.0. HTTP port 9592.

Components to wire (exactly):

- 1 AutonomousAgent PreAuthAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/pre-auth-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the VALIDATE, REVIEW, and DETERMINE tool sets are ALL registered on the
  agent; phase gating is the job of PhiSanitizer (which also doubles as the phase-gate
  guardrail), NOT of conditional .tools(...) wiring. The before-tool-call guardrail
  (PhiSanitizer) is registered on the agent via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow PreAuthWorkflow per requestId with five steps (four mandatory + one conditional):
  * validateStep — emits ValidationStarted on the entity, then calls componentClient
    .forAutonomousAgent(PreAuthAgent.class, "agent-" + requestId).runSingleTask(
      TaskDef.instructions("MemberId: " + sanitizedMemberId + "\nProcedureCode: " +
      procedureCode + "\nPhase: VALIDATE")
        .metadata("requestId", requestId)
        .metadata("phase", "VALIDATE")
        .taskType(PreAuthTasks.VALIDATE_REQUEST)
    ). Reads forTask(taskId).result(VALIDATE_REQUEST) to get ValidationResult. Writes
    PreAuthEntity.recordValidation(validationResult). WorkflowSettings.stepTimeout 60s.
  * reviewStep — emits ReviewStarted, then runSingleTask with TaskDef.instructions
    (formatReviewContext(validationResult, procedureRequest)) and metadata.phase = "REVIEW",
    taskType REVIEW_POLICY. Writes PreAuthEntity.recordPolicyReview(policyReview).
    stepTimeout 60s.
  * determineStep — emits DetermineStarted, then runSingleTask with TaskDef.instructions
    (formatDetermineContext(policyReview, validationResult)) and metadata.phase = "DETERMINE",
    taskType DETERMINE_OUTCOME. Writes PreAuthEntity.recordDetermination(determination).
    stepTimeout 60s.
  * evalStep — runs the deterministic ClinicalCompletenessScorer over (determination,
    policyReview, validationResult) and writes PreAuthEntity.recordEvaluation(eval).
    stepTimeout 5s.
  * humanHoldStep — conditional; runs only when determination.outcome() == DENIED. Emits
    HumanApprovalRequested on entity; transitions status to PENDING_HUMAN_REVIEW. Waits for
    acknowledgement signal. stepTimeout PT72H (72 hours). On timeout writes
    EscalationRequired; on signal-received writes HumanApprovalReceived and finalizes.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(PreAuthWorkflow::error). The error step writes RequestFailed
  and ends.

- 1 EventSourcedEntity PreAuthEntity (one per requestId). State PreAuthRecord{requestId,
  memberInfo: Optional<MemberInfo>, procedureRequest: Optional<ProcedureRequest>,
  validationResult: Optional<ValidationResult>, policyReview: Optional<PolicyReview>,
  determination: Optional<Determination>, eval: Optional<EvalResult>, status: PreAuthStatus,
  createdAt: Instant, finishedAt: Optional<Instant>, sanitizationLog: List<SanitizationRecord>,
  phaseRejections: List<PhaseRejection>}.
  PreAuthStatus enum: SUBMITTED, VALIDATING, VALIDATED, REVIEWING, POLICY_REVIEWED,
  DETERMINING, DETERMINED, PENDING_HUMAN_REVIEW, EVALUATED, ESCALATED, FAILED.
  Events: RequestSubmitted{memberId, procedureCode, diagnosisCode, treatingPhysicianNpi,
  clinicalNotes}, ValidationStarted, RequestValidated{validationResult}, ReviewStarted,
  PolicyReviewed{policyReview}, DetermineStarted, DeterminationWritten{determination},
  EvaluationScored{eval}, HumanApprovalRequested{requestedAt}, HumanApprovalReceived{
  reviewerNotes, receivedAt}, EscalationRequired{reason}, SanitizationApplied{field,
  hashedToken, tool, sanitizedAt}, PhaseGuardrailRejected{phase, tool, reason},
  RequestFailed{reason}.
  Commands: submit, startValidation, recordValidation, startReview, recordPolicyReview,
  startDetermine, recordDetermination, recordEvaluation, requestHumanApproval,
  acknowledgeApproval, requireEscalation, recordSanitization, recordPhaseRejection, fail,
  getRequest. emptyState() returns PreAuthRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View PreAuthView with row type PreAuthRow that mirrors PreAuthRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes PreAuthEntity events. ONE
  query getAllRequests: SELECT * AS requests FROM preauth_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PreAuthEndpoint at /api with POST /auth-requests (body {memberId, procedureCode,
    diagnosisCode, treatingPhysicianNpi, clinicalNotes}; mints requestId; calls
    PreAuthEntity.submit(request); then starts PreAuthWorkflow with id "preauth-" + requestId;
    returns {requestId}), GET /auth-requests (list from getAllRequests, sorted newest-first),
    GET /auth-requests/{id} (one row), GET /auth-requests/sse (Server-Sent Events forwarded
    from the view's stream-updates), POST /auth-requests/{id}/acknowledge (body {reviewerNotes};
    calls PreAuthEntity.acknowledgeApproval; signals PreAuthWorkflow; returns {status}),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- PreAuthTasks.java declaring three Task<R> constants:
    VALIDATE_REQUEST = Task.name("Validate request").description("Check member eligibility
      and validate the procedure code / ICD-10 linkage").resultConformsTo(
      ValidationResult.class);
    REVIEW_POLICY = Task.name("Review policy").description("Match payer policy articles and
      evaluate each criterion against the submitted clinical evidence").resultConformsTo(
      PolicyReview.class);
    DETERMINE_OUTCOME = Task.name("Determine outcome").description("Build a rationale and
      classify the authorization outcome based on the policy review").resultConformsTo(
      Determination.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {VALIDATE, REVIEW, DETERMINE}. Each function-tool method is annotated
  with the constant phase (use a custom annotation if the SDK's @FunctionTool does not carry
  a phase field — the guardrail reads it from a parallel registry built at startup if so).

- ValidateTools.java — @FunctionTool checkMemberEligibility(String memberId) -> EligibilityCheck
  reading from src/main/resources/sample-data/members/*.json keyed by hashed memberId;
  @FunctionTool validateProcedureCode(String procedureCode, String icd10Code) ->
  ProcedureCheck reading from procedure-codes/*.json.

- ReviewTools.java — @FunctionTool matchPolicyArticles(String procedureCode, String
  diagnosisCode) -> List<PolicyArticle> from policies/*.json; @FunctionTool
  evaluateCriteria(List<PolicyArticle> articles, String clinicalNotes) -> CriteriaEvaluation
  (deterministic: criterion is met if its keyword appears in the sanitized clinicalNotes string,
  case-insensitive).

- DetermineTools.java — @FunctionTool buildRationale(CriteriaEvaluation criteriaEvaluation,
  List<PolicyArticle> articles) -> String (builds a rationale paragraph from the criteria met/
  unmet and article titles); @FunctionTool classifyOutcome(CriteriaEvaluation
  criteriaEvaluation) -> AuthOutcome (APPROVED if metCount == totalCount; DENIED if metCount
  < totalCount / 2; PENDING_ADDITIONAL_INFO otherwise).

- PhiSanitizer.java — implements the before-tool-call hook. Two responsibilities: (1) PHI
  stripping — scans call payload JSON for fields named memberId, dateOfBirth, patientName,
  ssn or values matching SSN pattern or date-adjacent-to-name heuristic; replaces each with
  sha256(rawValue + requestId).substring(0, 12); emits SanitizationApplied event per field.
  (2) Phase-gate — reads the candidate tool call's @FunctionTool.phase attribute and the
  current PreAuthEntity status by requestId (from TaskDef metadata), applies the accept
  matrix from Section 8, and either passes or returns
  Guardrail.reject("phase-violation: <tool> requires <precondition>, saw <status>"). On
  phase-gate reject ALSO calls PreAuthEntity.recordPhaseRejection. On sanitization-parse
  failure blocks entirely with reason "sanitization-parse-failure".

- ClinicalCompletenessScorer.java — pure deterministic logic (no LLM). Inputs: Determination,
  PolicyReview, ValidationResult. Outputs: EvalResult. Four checks: (1) procedure code
  coverage — every coveredProcedureCodes[i] in Determination equals validationResult
  .procedureCheck.procedureCode(); (2) policy citation — citedArticleIds non-empty AND every
  id in policyReview.matchedArticles[].articleId(); (3) rationale adequacy —
  rationale.split("\\s+").length >= (outcome == DENIED ? 40 : 15); (4) outcome validity —
  outcome is non-null and in AuthOutcome enum. Score 1-5 base 1 +1 per check. Rationale names
  the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9592 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/requests.jsonl with 5 seeded request lines covering
  the three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/members/*.json — member eligibility records keyed by
  hashed memberId (the sanitizer hashes at intake; ValidateTools reads by hash).
  src/main/resources/sample-data/procedure-codes/*.json — CPT code definitions with ICD-10
  linkage tables.
  src/main/resources/sample-data/policies/*.json — payer policy articles per procedure code.
  All static; deterministic across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, H1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.phi = true (member IDs, ICD-10
  codes, clinical notes), decisions.authority_level = binding (the determination is acted on
  by the payer), oversight.human_in_loop = true (denials always have a human hold),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "phi-leak-to-llm", "incorrect-policy-match",
  "phase-violation", "denial-without-human-review", "hallucinated-policy-citation";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/pre-auth-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Medical Pre-Authorization",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of request cards; right = selected-request detail with eligibility summary,
  procedure validation result, policy review panel, determination outcome chip, eval-score
  chip, sanitization-log strip, phase-rejection-log strip). Browser title exactly:
  <title>Akka Sample: Medical Pre-Authorization</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes:
    validate-request.json — 5 ValidationResult entries, each with valid EligibilityCheck and
      ProcedureCheck. One entry deliberately fails eligibility (activeEligibility = false) so
      J5 is testable. tool_calls: checkMemberEligibility + validateProcedureCode per entry.
    review-policy.json — 5 PolicyReview entries paired one-to-one with validate entries.
      Each with 1-3 matched PolicyArticles and a CriteriaEvaluation. One entry has
      metCount < totalCount/2 to trigger a DENIED outcome.
    determine-outcome.json — 5 Determination entries. Outcomes match the review entries.
      One DENIED entry triggers humanHoldStep. One entry cites a policy articleId absent
      from its paired PolicyReview — triggers E1 score 1 (J4).
- MockModelProvider.seedFor(requestId) makes per-request selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PreAuthAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PreAuthTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (validateStep 60s, reviewStep 60s, determineStep 60s, evalStep 5s, humanHoldStep PT72H,
  error 5s).
- Lesson 6: every nullable lifecycle field on PreAuthRecord is Optional<T>.
- Lesson 7: PreAuthTasks.java with VALIDATE_REQUEST, REVIEW_POLICY, DETERMINE_OUTCOME
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9592 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- Single-agent invariant: exactly ONE AutonomousAgent (PreAuthAgent). ClinicalCompletenessScorer
  is rule-based and does NOT make an LLM call.
- Sequential-pipeline invariant: all three tool sets are registered on the agent; PhiSanitizer
  is the runtime gate (both PHI and phase-gate). Do NOT conditionally register tools per task.
- Task dependency: validateStep writes ValidationResult onto the entity; reviewStep reads it
  to build the REVIEW task's instruction context; determineStep reads both. The agent is
  stateless across phases.
- Overview tab Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
