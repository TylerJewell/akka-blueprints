# SPEC — dual-llm-pdf-extract

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Dual-LLM PDF Extractor.
**One-line pitch:** Upload a PDF; two extraction agents — Claude and Gemini — each extract structured data independently, then a reconciler merges their outputs into one authoritative result with a field-level confidence map and a disagreement score.

## 2. What this blueprint demonstrates

The **debate-multi-perspective** coordination pattern applied to document operations: a Workflow fans the same sanitized PDF text out to two extraction agents running on different model families, then asks a reconciler agent to merge their independent outputs. Where the agents agree, confidence is high; where they disagree, the reconciler flags the field and the eval sampler records the disagreement as a signal. The blueprint also demonstrates two governance mechanisms — a **PII sanitizer** that redacts personal data from the extracted text before any agent reads it, and an **eval-event** sampler that scores cross-model agreement on every completed extraction to track systematic model drift over time.

## 3. User-facing flows

The user opens the App UI tab and submits a PDF (the UI accepts the file and extracts its text client-side via the FileReader API before POSTing).

1. The system creates an `Extraction` record in `INTAKE` and starts an `ExtractionWorkflow`.
2. A deterministic PII sanitizer redacts emails, phone numbers, government identifiers, and obvious names from the extracted text. Only the redacted text is persisted and passed downstream. The extraction moves to `EXTRACTING`.
3. `ClaudeExtractor` and `GeminiExtractor` run concurrently over the same redacted text. Each returns a typed `RawExtraction` containing the extracted fields and a per-field confidence list.
4. `ExtractionReconciler` receives both `RawExtraction` results, compares them field by field, and produces a `MergedExtraction { fields, confidenceMap, disagreementFields, reconciliationNotes }`.
5. The extraction moves to `RECONCILED`.
6. If either extractor times out after 60 seconds, the workflow short-circuits: the reconciler works from the single available extraction, and the extraction enters `DEGRADED` with `failureReason` naming the missing extractor.
7. Every five minutes, an eval sampler picks one reconciled extraction and asks a judge to score overall cross-model agreement on a 1–5 scale, attaching field-level notes.

A `PdfSimulator` (TimedAction) drips a sample PDF text every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ClaudeExtractor` | `AutonomousAgent` | Extracts structured fields from the sanitized PDF text using the Anthropic model. | `ExtractionWorkflow` | returns typed result to workflow |
| `GeminiExtractor` | `AutonomousAgent` | Extracts the same structured fields from the same text using the Google AI Gemini model. | `ExtractionWorkflow` | returns typed result to workflow |
| `ExtractionReconciler` | `AutonomousAgent` | Receives both raw extractions, compares them field by field, and produces a merged authoritative result with a confidence map. | `ExtractionWorkflow` | returns typed result to workflow |
| `EvalJudge` | `AutonomousAgent` | Scores how far apart the two raw extractions were, producing a cross-model agreement score. | `AgreementSampler` | — |
| `ExtractionWorkflow` | `Workflow` | Coordinates redaction, parallel extraction fan-out, reconciliation. | `ExtractionEndpoint`, `DocumentConsumer` | `ExtractionEntity` |
| `ExtractionEntity` | `EventSourcedEntity` | Holds the extraction's lifecycle (intake → extracting → reconciled / degraded). | `ExtractionWorkflow`, `AgreementSampler` | `ExtractionView` |
| `DocumentQueue` | `EventSourcedEntity` | Logs each submitted document for replay/audit. | `ExtractionEndpoint`, `PdfSimulator` | `DocumentConsumer` |
| `ExtractionView` | `View` | List-of-extractions read model. | `ExtractionEntity` events | `ExtractionEndpoint` |
| `DocumentConsumer` | `Consumer` | Listens to `DocumentQueue` events and starts a workflow per submission. | `DocumentQueue` events | `ExtractionWorkflow` |
| `PdfSimulator` | `TimedAction` | Drips a sample PDF text every 60 s. | scheduler | `DocumentQueue` |
| `AgreementSampler` | `TimedAction` | Samples one reconciled extraction every 5 minutes for agreement scoring. | scheduler | `EvalJudge`, `ExtractionEntity` |
| `ExtractionEndpoint` | `HttpEndpoint` | `/api/extractions/*` — submit, get, list, SSE. | — | `ExtractionView`, `DocumentQueue`, `ExtractionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record DocumentSubmission(String filename, String extractedText, String submittedBy) {}

record ExtractedField(String name, String value, double confidence) {}

record RawExtraction(String modelFamily, List<ExtractedField> fields,
                     String extractorNotes, Instant extractedAt) {}
    // modelFamily: CLAUDE | GEMINI

record FieldAgreement(String fieldName, boolean agreed,
                      String claudeValue, String geminiValue) {}

record MergedExtraction(List<ExtractedField> fields,
                        Map<String, Double> confidenceMap,
                        List<FieldAgreement> disagreementFields,
                        String reconciliationNotes,
                        int disagreementCount,
                        Instant reconciledAt) {}

record AgreementVerdict(int score, String rationale,
                        List<String> highDisagreementFields) {}   // score 1–5

record RedactionResult(String redactedText, int redactionCount) {}

record Extraction(
    String extractionId,
    String filename,
    ExtractionStatus status,
    Optional<String> redactedText,
    Optional<Integer> redactionCount,
    Optional<RawExtraction> claudeExtraction,
    Optional<RawExtraction> geminiExtraction,
    Optional<MergedExtraction> mergedExtraction,
    Optional<String> failureReason,
    Optional<Integer> agreementScore,
    Optional<String> agreementRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExtractionStatus { INTAKE, EXTRACTING, RECONCILED, DEGRADED }
```

The raw submission `extractedText` is never stored on `ExtractionEntity`; only `redactedText` is persisted, after the sanitizer runs.

### Events (on `ExtractionEntity`)

`ExtractionCreated`, `DocumentSanitized`, `ClaudeExtractionAttached`, `GeminiExtractionAttached`, `ExtractionReconciled`, `ExtractionDegraded`, `AgreementScored`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/extractions` — body `{ filename, extractedText, submittedBy? }` → `{ extractionId }`. Starts a workflow.
- `GET /api/extractions` — list all extractions. Optional `?status=INTAKE|EXTRACTING|RECONCILED|DEGRADED`.
- `GET /api/extractions/{id}` — one extraction.
- `GET /api/extractions/sse` — server-sent events stream of every extraction change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Dual-LLM PDF Extractor"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to upload a PDF (or paste text directly), live list of extractions with status pills, expand-row to see both raw extractions, the merged result, the redaction count, and the agreement score.

Browser title: `<title>Akka Sample: Dual-LLM PDF Extractor</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid state-diagram label colours and edge-label `overflow:visible` per Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`sanitizer`, flavor `pii`): the `ExtractionWorkflow` sanitize step runs a deterministic redactor over the submitted text before any extractor reads it. Emails, phone numbers, government identifiers, and obvious personal names are replaced with typed placeholders. Only the redacted text is persisted. The redaction count is recorded on the entity. System-level.
- **E1 — eval-event agreement sampler** (`on-decision-eval`): `AgreementSampler` (TimedAction) picks one reconciled extraction every 5 minutes and asks `EvalJudge` to score how far apart the Claude and Gemini extractions were, emitting an `AgreementScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `ClaudeExtractor` → `prompts/claude-extractor.md`. Extracts structured fields from PDF text, returning a `RawExtraction`.
- `GeminiExtractor` → `prompts/gemini-extractor.md`. Extracts the same structured fields from the same text, independently, returning a `RawExtraction`.
- `ExtractionReconciler` → `prompts/extraction-reconciler.md`. Compares the two raw extractions field by field and produces a `MergedExtraction`.
- `EvalJudge` → `prompts/eval-judge.md`. Scores overall cross-model agreement, returning an `AgreementVerdict`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a PDF text; extraction progresses INTAKE → EXTRACTING → RECONCILED within 60 s; the expanded view shows both raw extractions and one merged result; UI reflects each transition via SSE.
2. **J2** — Submit text containing an email and a phone number; the persisted extraction has those redacted, the raw values appear nowhere in `/api/extractions/{id}`, and `redactionCount` is at least 2.
3. **J3** — Inject an extractor timeout (set one extractor's step timeout to 1 s); extraction enters DEGRADED with the merged result built from the single available extraction and `failureReason` naming the missing extractor.
4. **J4** — Wait one eval interval after a successful reconciliation; the extraction row shows an agreement score (1–5) and rationale.
5. **J5** — Two documents that differ substantially in content show different `disagreementCount` values, confirming per-document field-level tracking.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named dual-llm-pdf-extract demonstrating the
debate-multi-perspective × ops-automation cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
debate-multi-perspective-ops-automation-dual-llm-pdf-extract.
Java package io.akka.samples.extractandprocesspdfinfowithclaudeandgemini.
Akka 3.6.0. HTTP port 9445.

Components to wire (exactly):
- 4 AutonomousAgents:
  * ClaudeExtractor — definition() with capability(TaskAcceptance.of(EXTRACT_CLAUDE)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/claude-extractor.md.
    Returns RawExtraction{modelFamily="CLAUDE", fields: List<ExtractedField{name,
    value, confidence}>, extractorNotes, extractedAt}.
  * GeminiExtractor — capability(TaskAcceptance.of(EXTRACT_GEMINI)
    .maxIterationsPerTask(2)). System prompt from prompts/gemini-extractor.md.
    Returns RawExtraction{modelFamily="GEMINI", ...}.
  * ExtractionReconciler — capability(TaskAcceptance.of(RECONCILE)
    .maxIterationsPerTask(3)). System prompt from prompts/extraction-reconciler.md.
    Returns MergedExtraction{fields: List<ExtractedField>, confidenceMap:
    Map<String,Double>, disagreementFields: List<FieldAgreement{fieldName,
    agreed, claudeValue, geminiValue}>, reconciliationNotes, disagreementCount,
    reconciledAt}.
  * EvalJudge — capability(TaskAcceptance.of(SCORE_AGREEMENT)
    .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md.
    Returns AgreementVerdict{score, rationale, highDisagreementFields:
    List<String>}.

- 1 deterministic helper PiiSanitizer (plain class in application/, NOT an
  agent): method redact(String text) -> RedactionResult{redactedText,
  redactionCount}. Replaces emails, phone numbers, government identifiers
  (SSN-like, passport-like), and capitalised two-token name patterns with
  typed placeholders ([EMAIL], [PHONE], [ID], [NAME]). Pure, no LLM call.

- 1 Workflow ExtractionWorkflow with steps:
  createStep -> sanitizeStep -> [parallel] claudeStep, geminiStep ->
  joinStep -> reconcileStep -> emitStep.
  createStep calls ExtractionEntity.createExtraction emitting ExtractionCreated
  (INTAKE).
  sanitizeStep calls PiiSanitizer.redact(rawText), then
  ExtractionEntity.attachSanitized(redactedText, redactionCount) emitting
  DocumentSanitized (status -> EXTRACTING). The raw text is held only in the
  workflow's transient state and never persisted.
  claudeStep calls forAutonomousAgent(ClaudeExtractor.class, EXTRACT_CLAUDE)
  with the redactedText. On return calls ExtractionEntity.attachClaude(raw)
  emitting ClaudeExtractionAttached.
  geminiStep calls forAutonomousAgent(GeminiExtractor.class, EXTRACT_GEMINI)
  with the redactedText. On return calls ExtractionEntity.attachGemini(raw)
  emitting GeminiExtractionAttached.
  claudeStep and geminiStep run in parallel (CompletionStage zip of two calls);
  each wrapped with WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(60)).
  On any extractor timeout, transition to degradeStep that calls reconcileStep
  with whichever extraction returned, then ends with ExtractionDegraded
  (DEGRADED) and failureReason naming the missing extractor.
  reconcileStep calls forAutonomousAgent(ExtractionReconciler.class, RECONCILE)
  with the available RawExtraction(s). emitStep emits ExtractionReconciled
  (RECONCILED).
  Override settings() with stepTimeout(60s) on the two extractor steps and
  the reconcileStep, and defaultStepRecovery(maxRetries(2).failoverTo(error)).

- 1 EventSourcedEntity ExtractionEntity holding state Extraction{extractionId,
  filename, ExtractionStatus, Optional<String> redactedText, Optional<Integer>
  redactionCount, Optional<RawExtraction> claudeExtraction, Optional<RawExtraction>
  geminiExtraction, Optional<MergedExtraction> mergedExtraction, Optional<String>
  failureReason, Optional<Integer> agreementScore, Optional<String>
  agreementRationale, Instant createdAt, Optional<Instant> finishedAt}.
  ExtractionStatus enum: INTAKE, EXTRACTING, RECONCILED, DEGRADED.
  Events: ExtractionCreated, DocumentSanitized, ClaudeExtractionAttached,
  GeminiExtractionAttached, ExtractionReconciled, ExtractionDegraded,
  AgreementScored. Commands: createExtraction, attachSanitized, attachClaude,
  attachGemini, reconcile, degrade, recordAgreement, getExtraction. emptyState()
  returns Extraction.initial("", "") with no commandContext() reference.

- 1 EventSourcedEntity DocumentQueue with command enqueueDocument(filename,
  extractedText, submittedBy) emitting DocumentReceived{extractionId, filename,
  extractedText, submittedBy, submittedAt}.

- 1 View ExtractionView with row type ExtractionRow (mirrors Extraction minus
  the heavy redactedText; keep field counts, merged field list, and the
  agreement score for the list view). Table updater consumes ExtractionEntity
  events. ONE query getAllExtractions SELECT * AS extractions FROM
  extraction_view. No WHERE status filter — caller filters client-side.

- 1 Consumer DocumentConsumer subscribed to DocumentQueue events; on
  DocumentReceived starts an ExtractionWorkflow with the extractionId as the
  workflow id, passing the raw text in the workflow start command (not the
  entity).

- 2 TimedActions:
  * PdfSimulator — every 60s, reads next line from
    src/main/resources/sample-events/pdf-submissions.jsonl and calls
    DocumentQueue.enqueueDocument.
  * AgreementSampler — every 5 minutes, queries ExtractionView.getAllExtractions,
    picks the oldest RECONCILED extraction without an agreementScore, calls
    forAutonomousAgent(EvalJudge.class, SCORE_AGREEMENT) with the two raw
    extractions and the merged extraction, then calls
    ExtractionEntity.recordAgreement(score, rationale).

- 2 HttpEndpoints:
  * ExtractionEndpoint at /api with POST /extractions, GET /extractions,
    GET /extractions/{id}, GET /extractions/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The list filters by status client-side from
    getAllExtractions.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 Bootstrap (service-setup) that schedules the two TimedActions on startup
  and fails fast with a clear message if the configured model-provider key
  reference does not resolve (never echoing key material).

Companion files:
- ExtractionTasks.java declaring four Task<R> constants: EXTRACT_CLAUDE
  (RawExtraction), EXTRACT_GEMINI (RawExtraction), RECONCILE (MergedExtraction),
  SCORE_AGREEMENT (AgreementVerdict).
- Domain records DocumentSubmission, ExtractedField, RawExtraction,
  FieldAgreement, MergedExtraction, AgreementVerdict, RedactionResult.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9445 and akka.javasdk.agent model-provider
  blocks for anthropic (claude-sonnet-4-6), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Note: ClaudeExtractor uses the anthropic provider; GeminiExtractor uses the
  googleai-gemini provider; ExtractionReconciler and EvalJudge use whichever
  single provider is configured at runtime (defaulting to anthropic if both
  are present).
- src/main/resources/sample-events/pdf-submissions.jsonl with 8 canned
  document texts (invoices, contracts, reports), at least 3 of which contain
  PII (email/phone/id) so the sanitizer is visibly exercised.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 sanitizer pii,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types (note: PII may transit the system but is redacted before
  extraction), capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/claude-extractor.md, gemini-extractor.md, extraction-reconciler.md,
  eval-judge.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Dual-LLM PDF Extractor",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live
  list with status pills; expand shows both raw extractions, merged result,
  redaction count, agreement score). Browser title exactly:
  <title>Akka Sample: Dual-LLM PDF Extractor</title>. No subtitle on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If both are set, configure
  ClaudeExtractor for anthropic and GeminiExtractor for googleai-gemini and
  proceed. If only one is set, configure both extractors to that provider and
  note in application.conf that the dual-model behavior requires both keys.
- If none is set, ask the user how to source the key(s), offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the Task<R> id.
  Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json:
    claude-extractor.json — 4–6 RawExtraction entries (modelFamily="CLAUDE",
      3–8 ExtractedField objects with plausible invoice/contract fields,
      extractorNotes).
    gemini-extractor.json — 4–6 RawExtraction entries (modelFamily="GEMINI",
      same field names but subtly different values on 1–3 fields per entry to
      simulate natural model variation).
    extraction-reconciler.json — 4–6 MergedExtraction entries (merged fields,
      confidenceMap, 0–3 FieldAgreement disagreement entries, reconciliationNotes).
    eval-judge.json — 4–6 AgreementVerdict entries (score 1–5, one-sentence
      rationale, 0–2 highDisagreementFields).
- A MockModelProvider.seedFor(extractionId) helper makes the selection
  deterministic per extraction id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent never silently downgraded to Agent.
- (Lesson 4) Workflow step timeouts set explicitly on every agent-calling step
  (60s on the two extractor steps and the reconcile step).
- (Lesson 6) Optional<T> for every nullable lifecycle field on the Extraction
  record and the View row record.
- (Lesson 7) AutonomousAgent requires the companion ExtractionTasks.java.
- (Lesson 8) Verify model names against the provider's current lineup.
- (Lesson 9) Run command is "/akka:build" (Claude Code).
- (Lesson 10) Port 9445 declared explicitly in application.conf.
- (Lesson 11) Source-platform metadata is corpus-internal.
- (Lesson 12) UI fits the 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier labels are descriptive ("Runs out of the box"),
  never T1/T2/T3/T4 and never the word "deferred".
- (Lesson 23) No competitor brand names anywhere in user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables.
- (Lesson 25) Never write the model-provider key value to disk.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute.
- Parallel extractor steps use CompletionStage zip, NOT sequential calls.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
