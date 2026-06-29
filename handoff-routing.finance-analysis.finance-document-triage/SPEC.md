# SPEC — finance-document-triage

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Finance Document Triage.
**One-line pitch:** A classifier agent examines incoming finance documents, applies two layers of regulated redaction, and hands each document to the correct downstream processor that owns resolution end-to-end — with a before-response guardrail and an inline eval scoring every routing decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which* processor should own the document, then transfers the same task identity to a downstream specialist that produces the final processing result. The downstream processor is responsible for the entire outcome; the classifier does not narrate or summarise. Four governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer before any LLM call. Names, account numbers, tax ids, and national identifier numbers are redacted. No agent ever sees raw personal identifiers.
- A **sector sanitizer** runs in a second Consumer pass. Financial-sector regulated fields — AML risk flags, raw credit scores, fund codes, and regulatory-threshold amounts — are redacted or masked before reaching an agent.
- A **before-agent-response guardrail** runs on the processor's draft result. It checks the draft against a policy rubric (no invented approval amounts, no fabricated account balances, no regulatory conclusions the processor is not authorised to make) and blocks the draft from publication when it violates.
- An **on-decision eval** fires every time the classifier emits a routing decision. A `ClassificationJudge` agent grades the decision against the sanitized document on a 1–5 rubric. The score and rationale are written back to the document record and surfaced in the UI.

The pattern is a single-branch fan-out: the workflow branches on the classifier's document type, and only the chosen processor is invoked. The other processors see no traffic for that document.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live document list. Every document displays its type chip, status pill, classification score, and (if processed) the published result summary.
2. `DocumentSimulator` (TimedAction) ticks every 30 s and inserts a new canned document from `sample-events/finance-documents.jsonl` into `DocumentQueue`.
3. For each new document: `PiiSanitizer` (Consumer) redacts PII, then `SectorSanitizer` (Consumer) redacts sector-regulated fields. The second Consumer registers a `DocumentEntity` and starts a `DocumentWorkflow`.
4. The workflow calls `ClassifierAgent`, gets a `ClassificationDecision { docType, confidence, reason }`, and emits `DocumentClassified` on the entity.
5. Branch on `docType`:
   - `INVOICE` → workflow calls `InvoiceProcessor` with the `PROCESS` task and waits for the typed `ProcessingResult`.
   - `LOAN_APPLICATION` → workflow calls `LoanApplicationProcessor` with the same `PROCESS` task.
   - `COMPLIANCE_REVIEW` → workflow calls `ComplianceReviewProcessor` with the same `PROCESS` task.
6. The processor's draft `ProcessingResult` passes through the before-agent-response guardrail. If accepted, `ResultPublished` is emitted (terminal `PROCESSED`). If rejected, `ResultBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. Independent of the workflow, `ClassificationEvalScorer` (Consumer) listens for `DocumentClassified` events, calls `ClassificationJudge`, and writes `ClassificationScored { score, rationale }` back to the entity.
8. The user can click any document card and see the redacted payload, the classification reason, the classification score, the chosen processor, the draft (or blocked draft + violations), and the published result.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DocumentSimulator` | `TimedAction` | Drips simulated finance documents into `DocumentQueue` every 30 s. | scheduler | `DocumentQueue` |
| `DocumentQueue` | `EventSourcedEntity` | Append-only audit log of every inbound document (`InboundDocumentReceived`). | `DocumentSimulator`, `DocumentEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `DocumentQueue` events; redacts names, account numbers, tax ids, national ids. Passes redacted payload to `SectorSanitizer` via entity command. | `DocumentQueue` events | `DocumentEntity` |
| `SectorSanitizer` | `Consumer` | Subscribes to `DocumentEntity` events (on `DocumentPiiSanitized`); applies sector-regulated redaction (AML flags, credit scores, fund codes); registers the entity in `SECTOR_SANITIZED` state; starts a `DocumentWorkflow`. | `DocumentEntity` events | `DocumentEntity`, `DocumentWorkflow` |
| `ClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedDocument` into `INVOICE` / `LOAN_APPLICATION` / `COMPLIANCE_REVIEW` with confidence + reason. | invoked by `DocumentWorkflow` | returns `ClassificationDecision` |
| `InvoiceProcessor` | `AutonomousAgent` | Owns the `PROCESS` task for invoice documents. Returns typed `ProcessingResult`. | invoked by `DocumentWorkflow` | returns `ProcessingResult` |
| `LoanApplicationProcessor` | `AutonomousAgent` | Owns the `PROCESS` task for loan-application documents. Returns typed `ProcessingResult`. | invoked by `DocumentWorkflow` | returns `ProcessingResult` |
| `ComplianceReviewProcessor` | `AutonomousAgent` | Owns the `PROCESS` task for compliance-review documents. Returns typed `ProcessingResult`. | invoked by `DocumentWorkflow` | returns `ProcessingResult` |
| `ClassificationJudge` | `Agent` (typed) | Grades a classification decision against the sanitized document. Returns `ClassificationScore { score 1–5, rationale }`. | invoked by `ClassificationEvalScorer` | returns `ClassificationScore` |
| `OutputGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `ProcessingResult` against the policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `DocumentWorkflow` | returns `GuardrailVerdict` |
| `DocumentWorkflow` | `Workflow` | Per-document orchestration: classify → route → process → guardrail → publish. | `SectorSanitizer` (start) | `DocumentEntity` |
| `DocumentEntity` | `EventSourcedEntity` | Per-document lifecycle. | `DocumentWorkflow`, `ClassificationEvalScorer`, `PiiSanitizer`, `SectorSanitizer` | `DocumentView` |
| `DocumentView` | `View` | Read-model row per document. | `DocumentEntity` events | `DocumentEndpoint` |
| `ClassificationEvalScorer` | `Consumer` | Subscribes to `DocumentEntity` events; on `DocumentClassified` invokes `ClassificationJudge` and writes `ClassificationScored` back. | `DocumentEntity` events | `DocumentEntity` |
| `DocumentEndpoint` | `HttpEndpoint` | `/api/documents/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `DocumentView`, `DocumentEntity`, `DocumentQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI. | — | static resources |

## 5. Data model

```java
record IncomingDocument(
    String documentId,
    String sourceId,           // opaque source-system reference
    String docFormat,          // "pdf" | "xml" | "csv" | "json"
    String rawContent,         // raw text or structured payload as string
    String subject,            // human-readable document description
    Instant receivedAt
) {}

record SanitizedDocument(
    String redactedContent,
    String redactedSubject,
    List<String> piiCategoriesFound,       // ["name","account-number","tax-id","national-id"]
    List<String> sectorFieldsRedacted      // ["aml-flag","credit-score","fund-code","regulatory-amount"]
) {}

enum DocumentType { INVOICE, LOAN_APPLICATION, COMPLIANCE_REVIEW }

record ClassificationDecision(
    DocumentType docType,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum ProcessingAction {
    APPROVED, FLAGGED_FOR_REVIEW, INFORMATION_REQUESTED,
    REJECTED, FORWARDED_TO_COMPLIANCE, ESCALATED
}

record ProcessingResult(
    String resultSummary,
    String resultDetail,
    ProcessingAction action,
    String processorTag,       // "invoice" | "loan" | "compliance"
    Optional<String> referenceId,  // e.g. approval ref or review ticket id
    Instant processedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record ClassificationScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Document(
    String documentId,
    IncomingDocument incoming,
    Optional<SanitizedDocument> sanitized,
    Optional<ClassificationDecision> classification,
    Optional<ProcessingResult> result,
    Optional<GuardrailVerdict> guardrail,
    Optional<ClassificationScore> classificationScore,
    Optional<String> escalationReason,
    DocumentStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DocumentStatus {
    RECEIVED,
    PII_SANITIZED,
    SECTOR_SANITIZED,
    CLASSIFIED,
    ROUTED_INVOICE,
    ROUTED_LOAN,
    ROUTED_COMPLIANCE,
    RESULT_DRAFTED,
    BLOCKED,
    PROCESSED,
    ESCALATED
}
```

Events on `DocumentEntity`: `DocumentReceived`, `DocumentPiiSanitized`, `DocumentSectorSanitized`, `DocumentClassified`, `DocumentRouted`, `ResultDrafted`, `GuardrailVerdictAttached`, `ResultPublished`, `ResultBlocked`, `DocumentEscalated`, `ClassificationScored`.

Events on `DocumentQueue`: `InboundDocumentReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/documents` — list all documents (newest-first), optional `?docType=INVOICE|LOAN_APPLICATION|COMPLIANCE_REVIEW&status=…` filtered client-side.
- `GET /api/documents/{id}` — one document.
- `POST /api/documents` — manually submit a document (body `IncomingDocument` minus `documentId` and `receivedAt`); server assigns both.
- `POST /api/documents/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `PROCESSED` if the operator chooses to publish the blocked result.
- `GET /api/documents/sse` — Server-Sent Events for every document change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Finance Document Triage</title>`.

The App UI tab is a three-pane layout: **left** is the document list (status pill + type chip + score chip), **centre** is the selected document's redacted content + classification decision + score, **right** is the chosen processor's draft + guardrail verdict + published result (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts names, account numbers, tax ids, and national identifier numbers from the document content before any LLM sees it. The categories found are kept for audit.
- **S2 — Sector sanitizer** (`sector`, applied in `SectorSanitizer` Consumer): redacts AML risk flags, raw credit scores, fund codes, and regulatory-threshold amounts. These fields are regulated under financial-sector data-handling rules and must not be transmitted to a general-purpose model.
- **G1 — before-agent-response guardrail** on all three processors: checks every draft `ProcessingResult` against a rubric (no invented approval amounts, no fabricated account balances, no regulatory conclusions outside the processor's authority, no echoing of `[REDACTED]` tokens). Blocking — a violation puts the document in `BLOCKED` for operator review.
- **E1 — on-decision eval** (`eval-event`, on the classification decision): `ClassificationEvalScorer` (Consumer) listens for `DocumentClassified` events and calls `ClassificationJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate.

## 9. Agent prompts

- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier; returns one of `INVOICE`, `LOAN_APPLICATION`, `COMPLIANCE_REVIEW`; defaults to `COMPLIANCE_REVIEW` under ambiguity (conservative fallback that triggers human-assisted review).
- `InvoiceProcessor` → `prompts/invoice-processor.md`. Owns the `PROCESS` task for invoice documents. Never invents approval amounts; never invents payment-term interpretations.
- `LoanApplicationProcessor` → `prompts/loan-application-processor.md`. Owns the `PROCESS` task for loan applications. Never fabricates creditworthiness conclusions; escalates when outside authority.
- `ComplianceReviewProcessor` → `prompts/compliance-review-processor.md`. Owns the `PROCESS` task for compliance-review documents. Always forwards to a human compliance officer; never issues a regulatory conclusion autonomously.
- `ClassificationJudge` → `prompts/classification-judge.md`. Grades a classification decision against a 1–5 rubric with one-sentence rationale.
- `OutputGuardrail` → `prompts/output-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an invoice document → classified `INVOICE` → processed by `InvoiceProcessor` → guardrail passes → published.
2. **J2** — Simulator drips a loan-application document → classified `LOAN_APPLICATION` → processed by `LoanApplicationProcessor` → guardrail passes → published.
3. **J3** — A compliance-sensitive document is classified as `COMPLIANCE_REVIEW` → processed by `ComplianceReviewProcessor` → forwarded to compliance officer.
4. **J4** — A draft that cites an invented approval amount is blocked; the document lands in `BLOCKED` with the violation listed; the operator can unblock or leave it.
5. **J5** — Every classified document carries a `ClassificationScore` (1–5) and rationale within ~10 s of the classification decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named finance-document-triage demonstrating the handoff-routing × finance-analysis cell.
Runs out of the box (in-process simulated inbound stream; no real document-ingestion integration).
Maven group io.akka.samples. Maven artifact handoff-routing-finance-analysis-finance-document-triage.
Java package io.akka.samples.documentagentsfinancetriage. Akka 3.6.0. HTTP port 9715.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClassifierAgent — classifier. System prompt loaded from
  prompts/classifier-agent.md. Input: SanitizedDocument{redactedContent, redactedSubject,
  piiCategoriesFound: List<String>, sectorFieldsRedacted: List<String>}.
  Output: ClassificationDecision{docType: DocumentType (INVOICE/LOAN_APPLICATION/COMPLIANCE_REVIEW),
  confidence: "high"|"medium"|"low", reason: String}. Defaults to COMPLIANCE_REVIEW under
  uncertainty (conservative fallback).

- 1 AutonomousAgent InvoiceProcessor — definition() with capability(TaskAcceptance.of(PROCESS)
  .maxIterationsPerTask(3)). System prompt from prompts/invoice-processor.md.
  Input: SanitizedDocument + ClassificationDecision. Output: ProcessingResult{resultSummary,
  resultDetail, action: ProcessingAction, processorTag = "invoice",
  referenceId: Optional<String>, processedAt}. Never invents approval amounts or payment terms.

- 1 AutonomousAgent LoanApplicationProcessor — definition() with capability(TaskAcceptance.of(PROCESS)
  .maxIterationsPerTask(3)). System prompt from prompts/loan-application-processor.md.
  Same input shape; processorTag = "loan". Never fabricates creditworthiness conclusions.

- 1 AutonomousAgent ComplianceReviewProcessor — definition() with capability(TaskAcceptance.of(PROCESS)
  .maxIterationsPerTask(3)). System prompt from prompts/compliance-review-processor.md.
  Same input shape; processorTag = "compliance". Always sets action=FORWARDED_TO_COMPLIANCE
  and refers to a human officer; never issues regulatory conclusions autonomously.

- 1 Agent (typed) ClassificationJudge — judge. System prompt from prompts/classification-judge.md.
  Input: SanitizedDocument + ClassificationDecision. Output: ClassificationScore{score: int 1–5,
  rationale: String, scoredAt: Instant}.

- 1 Agent (typed) OutputGuardrail — typed rubric check. System prompt from
  prompts/output-guardrail.md. Input: SanitizedDocument + ProcessingResult. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by DocumentWorkflow before publishing; blocking.

- 1 Workflow DocumentWorkflow per documentId. Steps:
    classifyStep -> routeStep -> {invoiceStep | loanStep | complianceStep}
                -> guardrailStep -> publishStep
  classifyStep calls componentClient.forAgent().inSession(documentId)
    .method(ClassifierAgent::classify).invoke(sanitized).
    On success emits DocumentClassified via DocumentEntity.recordClassification.
  routeStep branches on ClassificationDecision.docType:
    INVOICE -> proceed to invoiceStep (emits DocumentRouted{INVOICE})
    LOAN_APPLICATION -> proceed to loanStep (emits DocumentRouted{LOAN_APPLICATION})
    COMPLIANCE_REVIEW -> proceed to complianceStep (emits DocumentRouted{COMPLIANCE_REVIEW})
  invoiceStep / loanStep / complianceStep call forAutonomousAgent(<Processor>.class, documentId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, classification))) returning a taskId,
    then forTask(taskId).result(FinanceTasks.PROCESS) to block on the typed ProcessingResult.
    On success emits ResultDrafted.
  guardrailStep calls forAgent(...).method(OutputGuardrail::check).invoke(sanitized, draft).
    On verdict.allowed=true proceed to publishStep (emits ResultPublished, terminal PROCESSED).
    On verdict.allowed=false emit ResultBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on invoiceStep, loanStep,
    complianceStep, and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(
    DocumentWorkflow::error)).

- 2 EventSourcedEntities:
    * DocumentQueue — append-only audit log. Command receive(IncomingDocument) emits
      InboundDocumentReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.documentId.
    * DocumentEntity (one per documentId) — full per-document lifecycle. State Document{documentId,
      incoming: IncomingDocument, Optional<SanitizedDocument> sanitized,
      Optional<ClassificationDecision> classification, Optional<ProcessingResult> result,
      Optional<GuardrailVerdict> guardrail, Optional<ClassificationScore> classificationScore,
      Optional<String> escalationReason, DocumentStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. DocumentStatus enum: RECEIVED, PII_SANITIZED,
      SECTOR_SANITIZED, CLASSIFIED, ROUTED_INVOICE, ROUTED_LOAN, ROUTED_COMPLIANCE,
      RESULT_DRAFTED, BLOCKED, PROCESSED, ESCALATED.
      Events: DocumentReceived, DocumentPiiSanitized, DocumentSectorSanitized,
      DocumentClassified, DocumentRouted, ResultDrafted, GuardrailVerdictAttached,
      ResultPublished, ResultBlocked, DocumentEscalated, ClassificationScored.
      Commands: registerIncoming, attachPiiSanitized, attachSectorSanitized,
      recordClassification, recordRouting, recordDraft, recordGuardrailVerdict, publish,
      block, escalate, recordClassificationScore, unblock, getDocument.
      emptyState() returns Document.initial("") with no commandContext() reference.

- 3 Consumers:
    * PiiSanitizer subscribed to DocumentQueue events; for each InboundDocumentReceived
      applies regex redaction pipeline (names via NER-pattern heuristics, account numbers
      matching ACCT-\w+ and \d{8,17}, tax ids matching [A-Z]{2}\d{6,12}, national ids
      matching NID-\w+) to rawContent + subject, builds intermediate redacted payload,
      calls DocumentEntity.registerIncoming then attachPiiSanitized. Does not start
      the workflow — that is SectorSanitizer's responsibility.
    * SectorSanitizer subscribed to DocumentEntity events; on DocumentPiiSanitized
      applies sector-regulated redaction (AML risk flags matching AML-\w+, credit scores
      matching FICO:\d+ or SCORE:\d+, fund codes matching FUND-\w+, regulatory amounts
      matching REG-AMT:\d+), builds SanitizedDocument with both piiCategoriesFound and
      sectorFieldsRedacted, calls DocumentEntity.attachSectorSanitized, then starts a
      DocumentWorkflow with documentId as the workflow id.
    * ClassificationEvalScorer subscribed to DocumentEntity events; on DocumentClassified
      invokes ClassificationJudge.score(sanitized, decision) and calls
      DocumentEntity.recordClassificationScore(documentId, score). On any other event
      type, no-op. Use componentClient — do NOT call the agent from a TimedAction.

- 1 View DocumentView with row type DocumentRow (mirrors Document; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes DocumentEntity events.
  ONE query getAllDocuments SELECT * AS documents FROM document_view. No WHERE docType or
  WHERE status filter — filter client-side in callers.

- 1 TimedAction DocumentSimulator — every 30s, reads next line from
  src/main/resources/sample-events/finance-documents.jsonl (loops at EOF) and calls
  DocumentQueue.receive with a fresh documentId (UUID).

- 2 HttpEndpoints:
    * DocumentEndpoint at /api with GET /documents (list from DocumentView.getAllDocuments,
      filter client-side by ?docType and ?status query params), GET /documents/{id},
      POST /documents (body IncomingDocument minus documentId/receivedAt — server assigns),
      POST /documents/{id}/unblock (body {decidedBy, note} — operator override:
      publishes the blocked result as PROCESSED with an audit note),
      GET /documents/sse (serverSentEventsForView over getAllDocuments), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- FinanceTasks.java declaring the task constants: PROCESS (resultConformsTo ProcessingResult.class,
  description "Process the finance document end-to-end and return a typed ProcessingResult").
- Domain records IncomingDocument, SanitizedDocument, ClassificationDecision, ProcessingResult,
  GuardrailVerdict, ClassificationScore, and the Document entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9715 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/finance-documents.jsonl with 9 canned lines (3 invoice-
  flavoured, 3 loan-application-flavoured, 2 compliance-review-flavoured, 1 designed to
  trip the guardrail with an invented approval amount).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls: S1 sanitizer pii, S2 sanitizer sector,
  G1 guardrail before-agent-response policy-rubric, E1 eval-event on-decision-eval. Matching
  simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = document-processing,
  data.data_classes.pii = true, data.data_classes.financial-data = true,
  decisions.authority_level = draft-only, oversight.human_in_loop = false (the system
  publishes without HITL by default — only blocked drafts wait for a human),
  data.pii_handled_by_sanitizer_before_llm = true, data.sector_data_redacted_before_llm = true,
  failure.failure_modes including "wrong-document-type-routing", "fabricated-approval-amount",
  "pii-leakage-via-llm", "sector-data-leakage", "regulatory-conclusion-overreach";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/classifier-agent.md, prompts/invoice-processor.md,
  prompts/loan-application-processor.md, prompts/compliance-review-processor.md,
  prompts/classification-judge.md, prompts/output-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Finance Document Triage", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = document list with type chip + status pill + score chip; centre = redacted
  content + classification block; right = processor draft + guardrail verdict + published
  result or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Finance Document Triage</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by documentId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    classifier-agent.json — 12 ClassificationDecision entries spanning
      INVOICE (vendor invoices, utility bills, purchase orders),
      LOAN_APPLICATION (mortgage applications, personal loan requests,
      business credit applications), and COMPLIANCE_REVIEW (AML alerts,
      sanctions checks, regulatory filings). Confidence + a one-sentence
      reason on each.
    invoice-processor.json — 8 ProcessingResult entries: 5 with action
      APPROVED or INFORMATION_REQUESTED (well within authority), 1 with
      action FLAGGED_FOR_REVIEW, 1 with action ESCALATED (amount exceeds
      threshold), 1 designed to trip the guardrail (cites an invented
      approval reference number).
    loan-application-processor.json — 8 ProcessingResult entries: 5 with
      INFORMATION_REQUESTED or FLAGGED_FOR_REVIEW, 1 with FORWARDED_TO_COMPLIANCE,
      1 with ESCALATED, 1 designed to trip the guardrail (invents a credit-
      score-based conclusion).
    compliance-review-processor.json — 8 ProcessingResult entries: all with
      action FORWARDED_TO_COMPLIANCE and a referenceId. One designed to trip
      the guardrail (issues an autonomous regulatory conclusion rather than
      forwarding).
    classification-judge.json — 10 ClassificationScore entries, score 1–5,
      one-sentence rationale (category-correctness / confidence-calibration /
      reason-quality).
    output-guardrail.json — 10 GuardrailVerdict entries. 7 with allowed=true
      and empty violations. 3 with allowed=false: "invented-approval-amount",
      "fabricated-credit-conclusion", "autonomous-regulatory-conclusion".
- A MockModelProvider.seedFor(documentId) helper makes per-document selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  InvoiceProcessor, LoanApplicationProcessor, and ComplianceReviewProcessor all
  extend akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): classifyStep 20s,
  guardrailStep 20s, invoiceStep / loanStep / complianceStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Document is Optional<T>. The
  DocumentView row type uses the same Optional wrapping.
- (Lesson 7) FinanceTasks.java declares the PROCESS Task<ProcessingResult> constant.
  All three processors' definition().capability(TaskAcceptance.of(PROCESS)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9715 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The PiiSanitizer and SectorSanitizer run INSIDE Consumers before any LLM call
  — not inside an Agent's prompt and not after the LLM has seen the raw payload.
  PiiSanitizer runs first; SectorSanitizer runs second on the entity's event.
- The ClassificationEvalScorer Consumer reacts to DocumentClassified events and
  calls ClassificationJudge via componentClient.forAgent(). It does NOT modify the
  workflow flow — the eval is out-of-band metadata.
- The guardrail step happens BEFORE ResultPublished. A blocked draft never reaches
  the UI as published — only as a "blocked draft + violations" surface for the operator.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
