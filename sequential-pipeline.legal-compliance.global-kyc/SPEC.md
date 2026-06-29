# SPEC — global-kyc-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Global KYC Agent.
**One-line pitch:** A compliance officer submits an applicant profile; one `KycAgent` walks it through three task phases — **COLLECT** identity documents, **VERIFY** them against jurisdiction rules, **DECIDE** a KYC outcome — with each phase gated on the prior phase's recorded output, PII sanitised before leaving the system boundary, and adverse decisions held for human review.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a legal-compliance domain. One `KycAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the COLLECT task's typed output (`DocumentSet`) becomes the VERIFY task's instruction context; the VERIFY task's typed output (`VerificationResult`) becomes the DECIDE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **PII sanitizer** runs as a `before-response` hook on the view-projection path. It receives the raw `DocumentSet` produced by the COLLECT phase — which carries full name, date of birth, document number, and issuing jurisdiction — and replaces each sensitive field with a stable token (e.g., `DOC-****-7843`) before writing to `KycView`. The entity event log retains the original values under its own access control. The sanitizer ensures that the read model, SSE stream, and UI card never expose raw PII, regardless of what the COLLECT phase produced.
- A **`hitl` application gate** is triggered by `KycPipelineWorkflow.decideStep` when `KycDecision.outcome == DECLINE`. The workflow emits `ReviewRequested`, pauses, and waits for a compliance officer to POST to `/api/cases/{id}/review` with a resolution (`APPROVED_AS_SUBMITTED` or `OVERRIDDEN_TO_PASS`). The agent does not run again during review — the HITL is a workflow-level pause, not an agent-level retry. A PASS decision proceeds to `evalStep` without triggering the gate.
- An **`on-decision-eval`** runs immediately after `DecisionRendered` lands, as `evalStep` inside the workflow. A deterministic, rule-based `DecisionScorer` (no LLM call — the eval is rule-based on purpose, so the same decision always scores the same) checks that the decision cites at least two jurisdiction rules by rule ID, that every cited rule ID exists in the loaded jurisdiction rule registry, that the `VerificationResult` has at least one document with `status == VERIFIED`, and that the `KycDecision.outcome` is one of the four valid values.

The blueprint shows that a sequential pipeline in a regulated domain adds governance at the right cut: PII is contained at the data-path level, adverse outcomes are held at the workflow step level, and completeness is measured at the decision-output level — each mechanism independent of the others.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects an **applicant profile** from the seeded list (or fills in the free-form fields: full name, date of birth, document type, document number, issuing jurisdiction).
2. The user clicks **Start KYC**. The UI POSTs to `/api/cases` and receives a `caseId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `COLLECTING` — the workflow has started `collectStep` and the agent has been handed the COLLECT task.
4. Within ~10–20 s the card reaches `VERIFYING` — the `DocumentSet` is visible in the card detail (a small table of documents with tokenised identifiers, document type, issuing jurisdiction, and fetch status). The VERIFY task has started.
5. Within ~10–20 s more the card reaches `DECIDING`. The `VerificationResult` is visible (document status list + rule-check outcomes).
6. If the outcome is PASS: within ~10–20 s the card reaches `DECIDED`, then `EVALUATED` within 1 s. The right pane shows the full typed `KycDecision` — outcome, jurisdiction, cited rules, rationale — plus an eval score chip (1–5) and a one-line scorer rationale.
7. If the outcome is DECLINE: the card reaches `PENDING_REVIEW`. A compliance officer reviews the detail pane and clicks **Approve** or **Override to Pass**. After the POST the card resumes to `DECIDED` → `EVALUATED`.
8. The user can submit another profile; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `KycEndpoint` | `HttpEndpoint` | `/api/cases/*` — submit, list, get, review, SSE; serves `/api/metadata/*`. | — | `KycCaseEntity`, `KycView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `KycCaseEntity` | `EventSourcedEntity` | Per-case lifecycle: created → collecting → collected → verifying → verified → deciding → decided → evaluated. Also handles pending_review and review_resolved. Source of truth. | `KycEndpoint`, `KycPipelineWorkflow` | `KycView` |
| `KycPipelineWorkflow` | `Workflow` | One workflow per case. Steps: `collectStep` → `verifyStep` → `decideStep` → `evalStep`. `decideStep` forks to `reviewStep` when outcome is DECLINE. Each agent-calling step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `KycEndpoint` after `CREATED` | `KycAgent`, `KycCaseEntity` |
| `KycAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `KycTasks.java`: `COLLECT_DOCUMENTS` → `DocumentSet`, `VERIFY_IDENTITY` → `VerificationResult`, `RENDER_DECISION` → `KycDecision`. Each task is registered with the phase-appropriate function tools. | invoked by `KycPipelineWorkflow` | returns typed results |
| `CollectTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchDocument(applicantId, docType)` and `lookupJurisdictionRules(jurisdiction)`. Reads from `src/main/resources/sample-data/applicants/*.json` for deterministic offline output. | called from COLLECT task | returns `List<Document>`, `List<JurisdictionRule>` |
| `VerifyTools` | function-tools class | Implements `checkDocumentAuthenticity(document)` and `evaluateRule(rule, document)`. Pure in-memory checks against the fetched jurisdiction rules. | called from VERIFY task | returns `DocumentVerification` / `RuleCheckResult` |
| `DecideTools` | function-tools class | Implements `compileDecision(verifications, rules)` and `buildRationale(outcome, ruleResults)`. Produces the typed `KycDecision` fields. | called from DECIDE task | returns `KycDecision` fields |
| `PiiSanitizer` | `before-response` hook (registered on view projection path) | Intercepts the raw `DocumentSet` before writing to `KycView`. Replaces `fullName`, `dateOfBirth`, and `documentNumber` with stable tokens. Raw values remain in the entity event log only. | every view-projection write from `KycCaseEntity` | tokenised `KycViewRow` |
| `HitlReviewGate` | plain class (no Akka primitive — workflow-level pause) | Called by `decideStep` when outcome is DECLINE. Emits `ReviewRequested` on the entity. Workflow pauses at `PENDING_REVIEW` waiting for `KycEndpoint.submitReview` to resume it. | invoked by `decideStep` | pauses workflow; resumes on review POST |
| `DecisionScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `KycDecision`, `VerificationResult`, `DocumentSet`. Output: `ScorerResult{score, rationale}`. | called from `evalStep` | returns score |
| `KycView` | `View` | Read model: one row per case for the UI. Projects `KycCaseEntity` events; view rows carry only tokenised PII. | `KycCaseEntity` events | `KycEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Document(
    String documentId,
    String applicantId,
    DocumentType docType,
    String issuingJurisdiction,
    String fullName,          // raw PII — sanitized before view projection
    String dateOfBirth,       // raw PII
    String documentNumber,    // raw PII
    Instant fetchedAt
) {}

enum DocumentType { PASSPORT, NATIONAL_ID, DRIVERS_LICENSE, RESIDENCE_PERMIT }

record DocumentSet(
    String applicantId,
    List<Document> documents,
    List<JurisdictionRule> applicableRules,
    Instant collectedAt
) {}

record JurisdictionRule(
    String ruleId,
    String jurisdiction,
    String description,
    String requiredDocumentType   // may be null if rule applies to all doc types
) {}

record DocumentVerification(
    String documentId,
    DocumentStatus status,
    String statusReason
) {}

enum DocumentStatus { VERIFIED, EXPIRED, TAMPERED, UNREADABLE, MISSING }

record RuleCheckResult(
    String ruleId,
    boolean passed,
    String failureReason    // null when passed == true
) {}

record VerificationResult(
    List<DocumentVerification> documentVerifications,
    List<RuleCheckResult> ruleResults,
    Instant verifiedAt
) {}

record KycDecision(
    KycOutcome outcome,
    String jurisdiction,
    List<String> citedRuleIds,
    String rationale,
    Instant decidedAt
) {}

enum KycOutcome { PASS, DECLINE, REFER, PENDING_DOCUMENTS }

record ScorerResult(
    int score,         // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record ReviewResolution(
    String reviewerId,
    ReviewOutcome resolution,
    String notes,
    Instant resolvedAt
) {}

enum ReviewOutcome { APPROVED_AS_SUBMITTED, OVERRIDDEN_TO_PASS }

record KycCaseRecord(
    String caseId,
    String applicantId,
    Optional<DocumentSet> documents,
    Optional<VerificationResult> verification,
    Optional<KycDecision> decision,
    Optional<ScorerResult> eval,
    Optional<ReviewResolution> review,
    KycCaseStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum KycCaseStatus {
    CREATED, COLLECTING, COLLECTED, VERIFYING, VERIFIED,
    DECIDING, DECIDED, PENDING_REVIEW, REVIEWED, EVALUATED, FAILED
}
```

Events on `KycCaseEntity`: `CaseCreated`, `CollectStarted`, `DocumentsCollected`, `VerifyStarted`, `IdentityVerified`, `DecideStarted`, `DecisionRendered`, `ReviewRequested`, `ReviewResolved`, `DecisionEvaluated`, `SanitizerApplied`, `CaseFailed`.

Every nullable lifecycle field on the `KycCaseRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/cases` — body `{ applicantId, jurisdiction, documentTypes }` → `{ caseId }`.
- `GET /api/cases` — list all cases, newest-first.
- `GET /api/cases/{id}` — one case.
- `POST /api/cases/{id}/review` — body `{ reviewerId, resolution, notes }` → `{ caseId, status }`. Resumes a paused DECLINE workflow.
- `GET /api/cases/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Global KYC Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of KYC cases (status pill + applicant ID + age + outcome chip when decided) and a right pane with the selected case's detail — documents table (tokenised), verification results, decision card, eval score chip, and a HITL review panel when the case is `PENDING_REVIEW`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer (pii)**: `PiiSanitizer` is registered on the view-projection path and runs before every write to `KycView`. It receives the raw `DocumentSet` and replaces `fullName` (token: `NAME-<hash8>`), `dateOfBirth` (token: `DOB-<hash8>`), and `documentNumber` (token: `DOC-****-<last4>`) with stable deterministic tokens. The entity event log retains the originals; every downstream consumer (view, SSE, UI) sees only tokens. The sanitizer also emits a `SanitizerApplied{fieldsRedacted: List<String>}` event for audit so the operator can confirm PII containment without inspecting the raw values.
- **H1 — HITL application gate**: `KycPipelineWorkflow.decideStep` inspects `KycDecision.outcome`. When outcome is `DECLINE`, the step calls `HitlReviewGate.request(caseId)`, which writes `ReviewRequested` on the entity (status → `PENDING_REVIEW`). The workflow transitions to its `reviewStep`, which calls `componentClient.forWorkflow(KycPipelineWorkflow.class, workflowId).waitForInput()`. The workflow stays paused until `KycEndpoint.submitReview(caseId, resolution)` calls `componentClient.forWorkflow(...).input(resolution)`. On receipt the workflow writes `ReviewResolved` and advances to `evalStep`. A REFER or PENDING_DOCUMENTS outcome also routes to `reviewStep`; a PASS outcome bypasses the gate and goes directly to `evalStep`. The gate is a workflow-level pause — no additional LLM call is made during review.
- **E1 — `on-decision-eval`**: runs immediately after `DecisionRendered` (or after `ReviewResolved` if the case went through review) lands, as `evalStep` inside the workflow. `DecisionScorer` is deterministic and rule-based (no LLM call). Four checks, one point per check satisfied on a base of 1: (1) rule citation coverage — at least two `citedRuleIds` in the decision; (2) rule ID validity — every `citedRuleId` exists in `DocumentSet.applicableRules[].ruleId`; (3) document verification — at least one `DocumentVerification` has `status == VERIFIED`; (4) outcome validity — `KycDecision.outcome` is one of `{PASS, DECLINE, REFER, PENDING_DOCUMENTS}`. Emits `DecisionEvaluated{score:1..5, rationale}` with a rationale sentence naming the largest gap. Score ≤ 2 flags the case for manual inspection in the UI.

## 9. Agent prompts

- `KycAgent` → `prompts/kyc-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Compliance officer submits the seeded profile `CORP-001`; within 60 s the case reaches `EVALUATED` with a PASS outcome, ≥ 1 verified document, ≥ 2 cited rule IDs, and an eval score chip showing 5 on the card.
2. **J2** — A DECLINE-outcome case pauses at `PENDING_REVIEW`; the workflow does not advance until the compliance officer POSTs `/api/cases/{id}/review` with `resolution = APPROVED_AS_SUBMITTED`; after the POST the case reaches `EVALUATED` within ~5 s.
3. **J3** — A case whose mock-LLM trajectory produces a `KycDecision` citing a rule ID absent from `DocumentSet.applicableRules` is scored 1–2 with a rationale naming the invalid rule reference; the UI flags the card.
4. **J4** — The right pane for any case in `COLLECTING` or later never shows raw PII (`fullName`, `dateOfBirth`, `documentNumber` appear only as tokens); the SSE event stream likewise contains only tokenised values.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named global-kyc-agent demonstrating the sequential-pipeline x legal-compliance
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-legal-compliance-global-kyc. Java package
io.akka.samples.globalkycagent. Akka 3.6.0. HTTP port 9356.

Components to wire (exactly):

- 1 AutonomousAgent KycAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/kyc-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...)  — the
  COLLECT, VERIFY, and DECIDE tool sets are ALL registered on the agent; phase gating is the
  job of the per-phase tool precondition checks embedded in the workflow step metadata, NOT
  of conditional .tools(...) wiring.

- 1 Workflow KycPipelineWorkflow per caseId with five steps:
  * collectStep — emits CollectStarted on the entity, then calls componentClient
    .forAutonomousAgent(KycAgent.class, "agent-" + caseId).runSingleTask(
      TaskDef.instructions("ApplicantId: " + applicantId +
        "\nJurisdiction: " + jurisdiction +
        "\nPhase: COLLECT\nFetch applicant documents and applicable jurisdiction rules.")
        .metadata("caseId", caseId)
        .metadata("phase", "COLLECT")
        .taskType(KycTasks.COLLECT_DOCUMENTS)
    ). Reads result to get DocumentSet. Writes KycCaseEntity.recordDocuments(documentSet).
    WorkflowSettings.stepTimeout 60s.
  * verifyStep — emits VerifyStarted, then runSingleTask with TaskDef.instructions
    (formatVerifyContext(documentSet, jurisdiction)) and metadata.phase = "VERIFY",
    taskType VERIFY_IDENTITY. Writes KycCaseEntity.recordVerification(result). stepTimeout 60s.
  * decideStep — emits DecideStarted, then runSingleTask with TaskDef.instructions
    (formatDecideContext(verification, documentSet)) and metadata.phase = "DECIDE",
    taskType RENDER_DECISION. Reads KycDecision. If outcome == DECLINE (or REFER or
    PENDING_DOCUMENTS) calls HitlReviewGate.request(caseId) and writes ReviewRequested,
    then transitions to reviewStep. If outcome == PASS writes DecisionRendered and
    transitions directly to evalStep. stepTimeout 60s.
  * reviewStep — emits ReviewRequested (already written by decideStep on non-PASS paths),
    then calls componentClient.forWorkflow(KycPipelineWorkflow.class, workflowId)
    .waitForInput(). Pauses indefinitely. On receipt of ReviewResolution via
    KycEndpoint.submitReview, writes ReviewResolved, then transitions to evalStep.
    stepTimeout 0 (no timeout on human review). maxRetries 0 (no auto-retry for HITL).
  * evalStep — runs the deterministic DecisionScorer over (decision, verification, documents)
    and writes KycCaseEntity.recordEval(score). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(KycPipelineWorkflow::error). The error step writes
  CaseFailed and ends.

- 1 EventSourcedEntity KycCaseEntity (one per caseId). State KycCaseRecord{caseId,
  applicantId: Optional<String>, documents: Optional<DocumentSet>,
  verification: Optional<VerificationResult>, decision: Optional<KycDecision>,
  eval: Optional<ScorerResult>, review: Optional<ReviewResolution>,
  status: KycCaseStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  KycCaseStatus enum: CREATED, COLLECTING, COLLECTED, VERIFYING, VERIFIED,
  DECIDING, DECIDED, PENDING_REVIEW, REVIEWED, EVALUATED, FAILED.
  Events: CaseCreated{applicantId, jurisdiction}, CollectStarted, DocumentsCollected{documents},
  VerifyStarted, IdentityVerified{verification}, DecideStarted, DecisionRendered{decision},
  ReviewRequested{caseId}, ReviewResolved{review}, DecisionEvaluated{eval},
  SanitizerApplied{fieldsRedacted: List<String>}, CaseFailed{reason}.
  Commands: create, startCollect, recordDocuments, startVerify, recordVerification,
  startDecide, recordDecision, requestReview, recordReview, recordEval, recordSanitizer,
  fail, getCase. emptyState() returns KycCaseRecord.initial("") with all Optional fields
  as Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View KycView with row type KycViewRow that mirrors KycCaseRecord with PII fields
  replaced by token Strings. Table updater consumes KycCaseEntity events; PiiSanitizer is
  called inside the updater before writing view columns for DocumentSet. ONE query
  getAllCases: SELECT * AS cases FROM kyc_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * KycEndpoint at /api with:
    POST /cases (body {applicantId, jurisdiction, documentTypes}; mints caseId;
      calls KycCaseEntity.create(applicantId, jurisdiction); starts
      KycPipelineWorkflow with id "kyc-" + caseId; returns {caseId}),
    GET /cases (list from getAllCases, sorted newest-first),
    GET /cases/{id} (one row),
    POST /cases/{id}/review (body {reviewerId, resolution, notes}; validates resolution
      enum; calls componentClient.forWorkflow(...).input(ReviewResolution); returns
      {caseId, status}),
    GET /cases/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving YAML/MD from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- KycTasks.java declaring three Task<R> constants:
    COLLECT_DOCUMENTS = Task.name("Collect documents").description("Fetch applicant
      identity documents and the jurisdiction's applicable rule set").
      resultConformsTo(DocumentSet.class);
    VERIFY_IDENTITY = Task.name("Verify identity").description("Check each document
      for authenticity and evaluate every applicable jurisdiction rule").
      resultConformsTo(VerificationResult.class);
    RENDER_DECISION = Task.name("Render decision").description("Compile all verification
      results and rule checks into a KycDecision with outcome and cited rule IDs").
      resultConformsTo(KycDecision.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- KycPhase.java — enum {COLLECT, VERIFY, DECIDE}. Each function-tool method is annotated with
  the constant phase.

- CollectTools.java — @FunctionTool fetchDocument(String applicantId, DocumentType docType)
  -> Document reading from src/main/resources/sample-data/applicants/<applicantId>.json;
  @FunctionTool lookupJurisdictionRules(String jurisdiction) -> List<JurisdictionRule>
  reading from src/main/resources/sample-data/rules/<jurisdiction>.json.

- VerifyTools.java — @FunctionTool checkDocumentAuthenticity(Document doc)
  -> DocumentVerification (checks expiry date, format, and a deterministic hash-based
  tamper flag); @FunctionTool evaluateRule(JurisdictionRule rule, Document doc)
  -> RuleCheckResult (checks whether doc.docType satisfies rule.requiredDocumentType,
  and whether all required fields are present).

- DecideTools.java — @FunctionTool compileDecision(List<DocumentVerification> verifications,
  List<RuleCheckResult> ruleResults) -> KycOutcome (PASS if all docs VERIFIED and all
  mandatory rules passed, DECLINE if any mandatory rule failed, REFER if ambiguous,
  PENDING_DOCUMENTS if a required document type is missing);
  @FunctionTool buildRationale(KycOutcome outcome, List<RuleCheckResult> ruleResults)
  -> String (one-sentence rationale naming the decisive rule or document).

- PiiSanitizer.java — pure static utility. tokenizeName(String fullName) -> String
  returns "NAME-" + sha1(fullName).substring(0,8);
  tokenizeDob(String dob) -> String returns "DOB-" + sha1(dob).substring(0,8);
  tokenizeDocNumber(String num) -> String returns "DOC-****-" + num.substring(
  Math.max(0, num.length()-4)). Called inside KycView's table-updater before writing
  the DocumentSet columns. Also emits SanitizerApplied audit event listing the three
  field names that were redacted.

- HitlReviewGate.java — plain class with a single static method request(String caseId)
  that returns a WorkflowStep.Effect that pauses on waitForInput(). The effect is
  returned by decideStep when outcome is non-PASS. The gate does not make any network
  call — the pause is implemented by the Akka workflow engine.

- DecisionScorer.java — pure deterministic logic (no LLM). Inputs: KycDecision,
  VerificationResult, DocumentSet. Outputs: ScorerResult with score and rationale.
  Four checks, one point per check satisfied, starting from a base of 1:
  (1) rule citation coverage (citedRuleIds.size() >= 2);
  (2) rule ID validity (every citedRuleId exists in documentSet.applicableRules ruleId set);
  (3) document verification (at least one documentVerification.status == VERIFIED);
  (4) outcome validity (outcome is a valid KycOutcome enum value). Score range 1-5.
  Rationale names the largest gap (rule with the lowest ordinal that failed).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9356 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/applicants.jsonl with 5 seeded applicant profile lines
  covering the three surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/applicants/*.json — four files keyed by applicant ID
  (CORP-001, CORP-002, RETAIL-001, RETAIL-002), each carrying Document entries with
  deterministic content so CollectTools.fetchDocument returns the same list across restarts.

- src/main/resources/sample-data/rules/*.json — three files keyed by jurisdiction
  (GB, US-NY, EU-DORA), each carrying 4-8 JurisdictionRule entries.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, H1, E1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true (applicant
  identity documents are PII), decisions.authority_level = automated-with-human-override
  (DECLINE outcomes require human review), oversight.human_in_loop = true (compliance
  officer reviews every DECLINE), operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "expired-document", "tampered-document",
  "missing-jurisdiction-rules", "invalid-rule-citation", "pii-leak-to-view";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/kyc-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Global KYC Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of KYC case cards; right = selected-case detail with documents table
  tokenised, verification results, decision card, eval-score chip, HITL review panel for
  PENDING_REVIEW cases). Browser title exactly: <title>Akka Sample: Global KYC Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(caseId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    collect-documents.json — 4 DocumentSet entries, each with 2-3 Document items and
      4-8 JurisdictionRule items, tool_calls containing fetchDocument (one per doc type)
      + lookupJurisdictionRules calls. Plus 1 entry for the PENDING_DOCUMENTS path
      (missing a required document type).
    verify-identity.json — 4 VerificationResult entries paired one-to-one, each with
      document verifications (status VERIFIED for the happy path) and rule check results
      (all passed), tool_calls containing checkDocumentAuthenticity + evaluateRule.
      Plus 1 TAMPERED entry whose DocumentVerification carries status = TAMPERED
      (leading to a DECLINE decision in the decide phase).
    render-decision.json — 4 KycDecision entries. Three with outcome PASS and 2 cited
      rule IDs from the paired applicableRules. One with outcome DECLINE and cited rule
      IDs that include a rule ID absent from applicableRules (J3 hallucinated-rule path).
      One with outcome PENDING_DOCUMENTS.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. KycAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion KycTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout.
  collectStep 60s, verifyStep 60s, decideStep 60s, evalStep 5s, reviewStep no timeout,
  error 5s.
- Lesson 6: every nullable lifecycle field on KycCaseRecord is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: KycTasks.java with COLLECT_DOCUMENTS, VERIFY_IDENTITY, RENDER_DECISION
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9356 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block. Without these, state names render
  black-on-black and arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (KycAgent). The
  on-decision eval is rule-based (DecisionScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  workflow step metadata carries the phase tag used by the workflow to set instruction
  context per phase. No conditional tool registration per task.
- Task dependency is carried by typed task results: collectStep writes DocumentSet onto the
  entity, verifyStep reads it and builds the VERIFY task's instruction context from it,
  decideStep reads both. The agent is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
