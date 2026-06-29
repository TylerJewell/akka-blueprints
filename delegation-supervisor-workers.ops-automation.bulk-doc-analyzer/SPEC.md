# SPEC — bulk-doc-analyzer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** High-Volume Document Analyzer.
**One-line pitch:** Drop a batch of documents; a coordinator delegates field extraction and risk classification to two specialist agents in parallel, then merges their outputs into one clean, PII-sanitized document record.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, collects their typed results, and asks a third AutonomousAgent to merge them into a processed document record. The blueprint also demonstrates a **PII sanitizer** that scrubs sensitive fields before the record is persisted, and an **eval-event** sampler that scores the coordinator's merge decision for quality monitoring.

## 3. User-facing flows

The user opens the App UI tab and submits a document batch (one or more raw document strings) via the form.

1. The system creates a `Batch` record in `PENDING` and one `Document` record per item in `QUEUED`.
2. `BatchWorkflow` starts one sub-workflow per document. `BatchCoordinator` partitions the work: each document gets an extraction instruction and a classification instruction.
3. For each document, the workflow forks: `Extractor` and `Classifier` run concurrently. Each returns a typed payload.
4. `BatchCoordinator` merges the two payloads into a `ProcessedDocument { fields, classification, sanitizedText, piiRedactions }`. The PII sanitizer runs inline at this step.
5. The processed document is written to `DocumentEntity` with status `PROCESSED`. When all documents in the batch are done, `BatchEntity` moves to `COMPLETE`.
6. If either worker times out after 60 seconds for a document, the workflow short-circuits: `BatchCoordinator` merges from whichever side returned, and the document enters `DEGRADED`. The batch continues with remaining documents.

A `DocumentSimulator` (TimedAction) drips a sample document batch every 90 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BatchCoordinator` | `AutonomousAgent` | Partitions a batch into per-document work items; merges Extractor + Classifier outputs; runs PII sanitization. | `BatchWorkflow` | returns typed result to workflow |
| `Extractor` | `AutonomousAgent` | Pulls structured fields from raw document text. Returns `ExtractedFields`. | `BatchWorkflow` | — |
| `Classifier` | `AutonomousAgent` | Assigns a risk category and sensitivity level to the document. Returns `DocumentClassification`. | `BatchWorkflow` | — |
| `BatchWorkflow` | `Workflow` | Fans each document out to Extractor and Classifier in parallel; calls BatchCoordinator to merge; emits final commands. | `DocumentEndpoint`, `BatchRequestConsumer` | `DocumentEntity`, `BatchEntity` |
| `DocumentEntity` | `EventSourcedEntity` | Holds the lifecycle of one document (queued → processing → processed / degraded). | `BatchWorkflow` | `DocumentView` |
| `BatchEntity` | `EventSourcedEntity` | Tracks the aggregate status of a submission batch (pending → in-progress → complete / partial). | `BatchWorkflow` | `DocumentView` |
| `BatchQueue` | `EventSourcedEntity` | Logs each batch submission for replay and audit. | `DocumentEndpoint`, `DocumentSimulator` | `BatchRequestConsumer` |
| `DocumentView` | `View` | List-of-documents read model, joined with batch status. | `DocumentEntity` events, `BatchEntity` events | `DocumentEndpoint` |
| `BatchRequestConsumer` | `Consumer` | Listens to `BatchQueue` events and starts a `BatchWorkflow` per submission. | `BatchQueue` events | `BatchWorkflow` |
| `DocumentSimulator` | `TimedAction` | Drips a sample document batch every 90 s. | scheduler | `BatchQueue` |
| `QualitySampler` | `TimedAction` | Samples one processed document every 5 minutes for quality scoring; emits a `QualityScored` event. | scheduler | `DocumentEntity` |
| `DocumentEndpoint` | `HttpEndpoint` | `/api/documents/*` — submit, get, list, SSE. | — | `DocumentView`, `BatchQueue`, `DocumentEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record BatchSubmission(List<String> rawDocuments, String submittedBy) {}

record ExtractedFields(
    Optional<String> documentDate,
    Optional<String> author,
    Optional<String> referenceNumber,
    String bodyText,
    Instant extractedAt
) {}

record DocumentClassification(
    RiskCategory riskCategory,
    SensitivityLevel sensitivityLevel,
    List<String> flaggedTopics,
    Instant classifiedAt
) {}

record PiiRedaction(String fieldName, String piiType, String replacement) {}

record ProcessedDocument(
    ExtractedFields fields,
    DocumentClassification classification,
    String sanitizedText,
    List<PiiRedaction> piiRedactions,
    Instant processedAt
) {}

record Document(
    String documentId,
    String batchId,
    String rawContent,
    DocumentStatus status,
    Optional<ExtractedFields> fields,
    Optional<DocumentClassification> classification,
    Optional<ProcessedDocument> processed,
    Optional<String> failureReason,
    Optional<Integer> qualityScore,
    Optional<String> qualityRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

record Batch(
    String batchId,
    String submittedBy,
    BatchStatus status,
    int totalDocuments,
    int processedCount,
    int degradedCount,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DocumentStatus { QUEUED, PROCESSING, PROCESSED, DEGRADED }
enum BatchStatus   { PENDING, IN_PROGRESS, COMPLETE, PARTIAL }
enum RiskCategory  { LOW, MEDIUM, HIGH, CRITICAL }
enum SensitivityLevel { PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED }
```

### Events (on `DocumentEntity`)

`DocumentQueued`, `DocumentProcessingStarted`, `FieldsExtracted`, `DocumentClassified`, `DocumentProcessed`, `DocumentDegraded`, `QualityScored`.

### Events (on `BatchEntity`)

`BatchCreated`, `BatchStarted`, `BatchDocumentCompleted`, `BatchComplete`, `BatchPartiallyComplete`.

### Events (on `BatchQueue`)

`BatchSubmitted { batchId, submittedBy, documentCount, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/documents/batches` — body `{ rawDocuments: [...], submittedBy }` → `{ batchId }`. Starts a workflow per document.
- `GET /api/documents/batches` — list all batches with aggregate counts.
- `GET /api/documents/batches/{batchId}` — one batch with its documents.
- `GET /api/documents` — list all documents. Optional `?status=QUEUED|PROCESSING|PROCESSED|DEGRADED`.
- `GET /api/documents/{id}` — one document.
- `GET /api/documents/sse` — server-sent events stream of every document change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "High-Volume Document Analyzer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a document batch, live list of documents with status pills, expand-row to see extracted fields + classification + processed summary + quality score.

Browser title: `<title>Akka Sample: High-Volume Document Analyzer</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`before-agent-response` on `BatchCoordinator`): scans the merged `ProcessedDocument` for names, email addresses, national ID numbers, and phone numbers before the record is written to `DocumentEntity`. Blocking. Detected PII fields are redacted and logged as `PiiRedaction` entries.
- **E1 — eval-event sampler** (`on-decision-eval`): `QualitySampler` (TimedAction) picks one processed document every 5 minutes and emits a `QualityScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `BatchCoordinator` → `prompts/coordinator.md`. Partitions incoming batch; later merges Extractor and Classifier outputs into a `ProcessedDocument`.
- `Extractor` → `prompts/extractor.md`. Pulls structured fields; returns `ExtractedFields`.
- `Classifier` → `prompts/classifier.md`. Assigns risk category and sensitivity; returns `DocumentClassification`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a two-document batch; both documents progress QUEUED → PROCESSING → PROCESSED within 90 s; batch moves to COMPLETE; UI reflects each transition via SSE.
2. **J2** — Inject an Extractor timeout (set step timeout to 1 s); affected document enters DEGRADED with whichever partial output came back; batch continues and finishes PARTIAL.
3. **J3** — Inject a document with a synthetic SSN in the raw content; the processed record has the SSN replaced by a redaction placeholder; a `PiiRedaction` entry appears in the record.
4. **J4** — Wait after a successful PROCESSED document; the document's row in the UI shows a quality score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named bulk-doc-analyzer demonstrating the
delegation-supervisor-workers × ops-automation cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-ops-automation-bulk-doc-analyzer.
Java package io.akka.samples.highvolumedocumentanalyzer. Akka 3.6.0.
HTTP port 9593.

Components to wire (exactly):
- 3 AutonomousAgents:
  * BatchCoordinator — definition() with capability(TaskAcceptance.of(PARTITION).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(MERGE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/coordinator.md. Returns WorkPartition{extractionInstruction,
    classificationInstruction} for PARTITION and ProcessedDocument{fields, classification,
    sanitizedText, piiRedactions, processedAt} for MERGE.
  * Extractor — capability(TaskAcceptance.of(EXTRACT).maxIterationsPerTask(3)). System prompt
    from prompts/extractor.md. Returns ExtractedFields{documentDate, author, referenceNumber,
    bodyText, extractedAt}.
  * Classifier — capability(TaskAcceptance.of(CLASSIFY).maxIterationsPerTask(2)). System prompt
    from prompts/classifier.md. Returns DocumentClassification{riskCategory, sensitivityLevel,
    flaggedTopics, classifiedAt}.

- 1 Workflow BatchWorkflow with steps:
  partitionStep -> [parallel] extractStep, classifyStep -> joinStep -> mergeStep ->
  sanitizeStep -> emitStep.
  partitionStep calls forAutonomousAgent(BatchCoordinator.class, PARTITION).
  extractStep and classifyStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(BatchWorkflow::extractStep, ofSeconds(60)) and
  stepTimeout(BatchWorkflow::classifyStep, ofSeconds(60)). On either timeout, transition to
  degradeStep that calls mergeStep with whichever side returned, then ends with DocumentDegraded.
  mergeStep calls forAutonomousAgent(BatchCoordinator.class, MERGE) with the merged inputs;
  give mergeStep a 90s stepTimeout. sanitizeStep runs the PII scanner over the sanitizedText
  field, fills piiRedactions; on detection, replaces PII spans with [REDACTED:<piiType>] and
  logs each replacement as a PiiRedaction entry. WorkflowSettings is nested inside Workflow —
  no import.

- 2 EventSourcedEntities:
  * DocumentEntity holding state Document{documentId, batchId, rawContent, DocumentStatus,
    Optional<ExtractedFields>, Optional<DocumentClassification>, Optional<ProcessedDocument>,
    Optional<String> failureReason, Optional<Integer> qualityScore,
    Optional<String> qualityRationale, Instant createdAt, Optional<Instant> finishedAt}.
    DocumentStatus enum: QUEUED, PROCESSING, PROCESSED, DEGRADED.
    Events: DocumentQueued, DocumentProcessingStarted, FieldsExtracted, DocumentClassified,
    DocumentProcessed, DocumentDegraded, QualityScored.
    Commands: queueDocument, startProcessing, attachFields, attachClassification,
    markProcessed, markDegraded, recordQuality, getDocument.
    emptyState() returns Document.initial("", "", "") with no commandContext() reference.
  * BatchEntity holding state Batch{batchId, submittedBy, BatchStatus, totalDocuments,
    processedCount, degradedCount, Instant createdAt, Optional<Instant> finishedAt}.
    BatchStatus enum: PENDING, IN_PROGRESS, COMPLETE, PARTIAL.
    Events: BatchCreated, BatchStarted, BatchDocumentCompleted, BatchComplete,
    BatchPartiallyComplete.
    Commands: createBatch, startBatch, documentDone, getBatch.

- 1 EventSourcedEntity BatchQueue with command submitBatch(rawDocuments, submittedBy) emitting
  BatchSubmitted{batchId, submittedBy, documentCount, submittedAt}.

- 1 View DocumentView with row type DocumentRow (mirrors Document minus rawContent; every
  nullable field is Optional<T>). Table updater consumes DocumentEntity events. ONE query
  getAllDocuments SELECT * AS documents FROM document_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer BatchRequestConsumer subscribed to BatchQueue events; on BatchSubmitted, creates
  the BatchEntity and starts one BatchWorkflow per document in the batch, keyed by documentId.

- 2 TimedActions:
  * DocumentSimulator — every 90s, reads next line from
    src/main/resources/sample-events/sample-batches.jsonl and calls BatchQueue.submitBatch.
  * QualitySampler — every 5 minutes, queries DocumentView.getAllDocuments, picks the oldest
    PROCESSED document without a qualityScore, runs a 1–5 rubric judge over the processed
    content, then calls DocumentEntity.recordQuality(score, rationale).

- 2 HttpEndpoints:
  * DocumentEndpoint at /api/documents with POST /batches, GET /batches, GET /batches/{batchId},
    GET (list documents), GET /{id}, GET /sse, and the /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DocumentTasks.java declaring four Task<R> constants: PARTITION (WorkPartition), EXTRACT
  (ExtractedFields), CLASSIFY (DocumentClassification), MERGE (ProcessedDocument).
- Domain records WorkPartition, ExtractedFields, DocumentClassification, PiiRedaction,
  ProcessedDocument.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9593 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-batches.jsonl with 8 canned batch lines,
  each a JSON object with rawDocuments (1-3 strings) and submittedBy.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 PII sanitizer before-agent-response,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = ops-automation / document-
  processing, decisions.authority_level = automated-action, data.data_classes.pii = true,
  capabilities.content_generation = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/extractor.md, prompts/classifier.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: High-Volume Document Analyzer", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: High-Volume Document Analyzer</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
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
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  extractor.json, classifier.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either WorkPartition or ProcessedDocument objects.
      4–6 WorkPartition entries (extractionInstruction + classificationInstruction
      pairs) and 4–6 ProcessedDocument entries (each with non-empty sanitizedText,
      empty piiRedactions list, a valid ExtractedFields and DocumentClassification).
    extractor.json — 4–6 ExtractedFields entries, each with a documentDate
      (ISO-8601 date string), an author name, a referenceNumber in the format
      "REF-YYYY-NNNN", and bodyText of 30–80 words.
    classifier.json — 4–6 DocumentClassification entries, each with a RiskCategory
      (LOW/MEDIUM/HIGH/CRITICAL), a SensitivityLevel (PUBLIC/INTERNAL/CONFIDENTIAL/
      RESTRICTED), and 1–3 flaggedTopics strings.
- A MockModelProvider.seedFor(documentId) helper makes the selection
  deterministic per document id so the same document produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s merge); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion DocumentTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9593 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
