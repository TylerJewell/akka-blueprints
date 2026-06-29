# SPEC — schema-extractor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SchemaExtractor.
**One-line pitch:** A user submits an unstructured document and a target JSON schema; one AI agent reads the document (passed as a task attachment, never as inline prompt text) and returns a typed `ExtractedRecord` whose fields are validated against the schema before the record is persisted.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `ExtractionAgent` (AutonomousAgent) carries the entire extraction decision; the surrounding components only prepare its input and validate its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw document submission and the agent call — so the model never sees identifiers.
- An **after-llm-response guardrail** validates the agent's extracted record on every turn: well-formed JSON, all required fields present, field types match the declared schema, and no field exceeds its declared max length. A schema-invalid record triggers a retry inside the same task.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a document into the **Document** textarea (or picks one of three seeded examples — a vendor invoice, a purchase-order confirmation, and a support-ticket email thread).
2. The user picks a **target schema** from a dropdown (Invoice, PurchaseOrder, SupportTicket) or pastes a custom JSON schema definition.
3. The user clicks **Submit for extraction**. The UI POSTs to `/api/jobs` and receives a `jobId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted document is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `extractStep` completes. The card transitions to `EXTRACTING` then `RECORD_EXTRACTED`. The extracted record appears: a field-value table with type chips, a schema-coverage badge (how many required fields are populated), and a confidence indicator.
6. The user can submit another document; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ExtractionEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ExtractionJobEntity`, `ExtractionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ExtractionJobEntity` | `EventSourcedEntity` | Per-job lifecycle: submitted → sanitized → extracting → extracted → failed. Source of truth. | `ExtractionEndpoint`, `DocumentSanitizer`, `ExtractionWorkflow` | `ExtractionView` |
| `DocumentSanitizer` | `Consumer` | Subscribes to `DocumentSubmitted` events; redacts PII; calls `ExtractionJobEntity.attachSanitized`. | `ExtractionJobEntity` events | `ExtractionJobEntity` |
| `ExtractionWorkflow` | `Workflow` | One workflow per job. Steps: `awaitSanitizedStep` → `extractStep` → `error`. | started by `DocumentSanitizer` once sanitized event lands | `ExtractionAgent`, `ExtractionJobEntity` |
| `ExtractionAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the target schema in the task definition and the sanitized document as a task attachment; returns `ExtractedRecord`. | invoked by `ExtractionWorkflow` | returns record |
| `ExtractionView` | `View` | Read model: one row per job for the UI. | `ExtractionJobEntity` events | `ExtractionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record FieldSchema(
    String fieldName,
    String fieldType,          // "string" | "number" | "boolean" | "date"
    boolean required,
    int maxLength              // 0 = no limit
) {}

record TargetSchema(
    String schemaId,
    String schemaName,
    List<FieldSchema> fields
) {}

record ExtractionRequest(
    String jobId,
    String documentTitle,
    String rawDocument,
    TargetSchema targetSchema,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedDocument(
    String redactedDocument,
    List<String> piiCategoriesFound
) {}

record ExtractedField(
    String fieldName,
    String fieldType,
    String value,              // always serialized as String; caller casts
    FieldConfidence confidence
) {}
enum FieldConfidence { HIGH, MEDIUM, LOW, ABSENT }

record ExtractedRecord(
    String schemaId,
    List<ExtractedField> fields,
    int schemaCoveragePercent, // required fields populated / total required × 100
    Instant extractedAt
) {}

record ExtractionJob(
    String jobId,
    Optional<ExtractionRequest> request,
    Optional<SanitizedDocument> sanitized,
    Optional<ExtractedRecord> record,
    ExtractionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExtractionStatus {
    SUBMITTED, SANITIZED, EXTRACTING, RECORD_EXTRACTED, FAILED
}
```

Events on `ExtractionJobEntity`: `DocumentSubmitted`, `DocumentSanitized`, `ExtractionStarted`, `RecordExtracted`, `ExtractionFailed`.

Every nullable lifecycle field on the `ExtractionJob` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/jobs` — body `{ documentTitle, rawDocument, targetSchema: TargetSchema, submittedBy }` → `{ jobId }`.
- `GET /api/jobs` — list all jobs, newest-first.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SchemaExtractor</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted jobs (status pill + schema-coverage badge + age) and a right pane with the selected job's detail — target schema definition, sanitized document preview, extracted field table, confidence chips, and coverage badge.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `DocumentSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw document before any LLM call. Records which categories were found.
- **G1 — after-llm-response guardrail**: runs on every turn of `ExtractionAgent`. Asserts the candidate response is valid `ExtractedRecord` JSON, every `fields[].fieldName` matches a field declared in the submitted `TargetSchema`, every required field is present and non-empty, every `fieldType` matches the declared type, and no string field value exceeds its declared `maxLength`. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `ExtractionAgent` → `prompts/extraction-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached document, walk every field in the target schema, and return one `ExtractedField` per schema field with a value drawn from the document.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Invoice seed; within 30 s the extracted record appears with one field entry per schema field and a schema-coverage badge of 100%.
2. **J2** — The agent's first response on a job is intentionally malformed (mock LLM path) — the `after-llm-response` guardrail rejects it; the second iteration produces a valid record; the UI never displays the malformed response.
3. **J3** — A document containing `jane.doe@example.com` and `4111-1111-1111-1111` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PCN]`; the entity's `request.rawDocument` retains the raw text for audit.
4. **J4** — A document missing several required fields causes the guardrail to reject the first agent response; after retry the agent marks those fields with `confidence = ABSENT`; the record lands with a coverage badge below 100%.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named schema-extractor demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-schema-extractor. Java package
io.akka.samples.structuredextractionagent. Akka 3.6.0. HTTP port 9630.

Components to wire (exactly):

- 1 AutonomousAgent ExtractionAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/extraction-agent.md>) and
  .capability(TaskAcceptance.of(EXTRACT_RECORD).maxIterationsPerTask(3)). The task receives
  the target schema as its instruction text and the sanitized document as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: ExtractedRecord{schemaId: String, fields: List<ExtractedField>,
  schemaCoveragePercent: int, extractedAt: Instant}. The agent is configured with an
  after-llm-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow ExtractionWorkflow per jobId with three steps:
  * awaitSanitizedStep — polls ExtractionJobEntity.getJob every 1s; on job.sanitized().isPresent()
    advances to extractStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * extractStep — emits ExtractionStarted, then calls componentClient.forAutonomousAgent(
    ExtractionAgent.class, "extractor-" + jobId).runSingleTask(
      TaskDef.instructions(formatSchema(job.request.targetSchema))
        .attachment("document.txt", job.sanitized.redactedDocument.getBytes())
    ) — returns a taskId, then forTask(taskId).result(EXTRACT_RECORD) to fetch the record.
    On success calls ExtractionJobEntity.recordExtracted(record). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(ExtractionWorkflow::error).
  * error step transitions the entity to FAILED via ExtractionJobEntity.fail(reason).
    WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ExtractionJobEntity (one per jobId). State ExtractionJob{jobId: String,
  request: Optional<ExtractionRequest>, sanitized: Optional<SanitizedDocument>,
  record: Optional<ExtractedRecord>, status: ExtractionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. ExtractionStatus enum: SUBMITTED, SANITIZED, EXTRACTING,
  RECORD_EXTRACTED, FAILED. Events: DocumentSubmitted{request}, DocumentSanitized{sanitized},
  ExtractionStarted{}, RecordExtracted{record}, ExtractionFailed{reason}. Commands: submit,
  attachSanitized, markExtracting, recordExtracted, fail, getJob. emptyState() returns
  ExtractionJob.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer DocumentSanitizer subscribed to ExtractionJobEntity events; on DocumentSubmitted
  runs a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like, payment-card-like,
  postal addresses, person-name heuristic, account-id-like tokens) over rawDocument, computes
  the list of categories found, builds SanitizedDocument, then calls
  ExtractionJobEntity.attachSanitized(sanitized). After attachSanitized lands, the same
  Consumer starts an ExtractionWorkflow with id = "extraction-" + jobId.

- 1 View ExtractionView with row type ExtractionRow (mirrors ExtractionJob minus
  request.rawDocument — the audit log keeps the raw; the view holds the sanitized form for
  the UI). Table updater consumes ExtractionJobEntity events. ONE query getAllJobs:
  SELECT * AS jobs FROM extraction_view. No WHERE status filter — Akka cannot auto-index enum
  columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ExtractionEndpoint at /api with POST /jobs (body
    {documentTitle, rawDocument, targetSchema: {schemaId, schemaName, fields: [{fieldName,
    fieldType, required, maxLength}]}, submittedBy}; mints jobId; calls
    ExtractionJobEntity.submit; returns {jobId}), GET /jobs (list from getAllJobs,
    sorted newest-first), GET /jobs/{id} (one row), GET /jobs/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ExtractionTasks.java declaring one Task<R> constant: EXTRACT_RECORD = Task.name("Extract
  record").description("Read the attached document and produce an ExtractedRecord matching
  the target schema").resultConformsTo(ExtractedRecord.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records FieldSchema, TargetSchema, ExtractionRequest, SanitizedDocument,
  ExtractedField, FieldConfidence, ExtractedRecord, ExtractionJob, ExtractionStatus.

- RecordGuardrail.java implementing the after-llm-response hook. Reads the candidate
  ExtractedRecord from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9630 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ExtractionAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/schemas/ with three seeded target schemas as JSON:
  invoice-schema.json (10 fields: invoiceNumber, vendorName, vendorTaxId, issueDate,
  dueDate, currency, subtotal, taxAmount, totalAmount, paymentTerms — all required except
  taxAmount), purchase-order-schema.json (8 fields: poNumber, buyerName, supplierName,
  orderDate, deliveryDate, lineItemCount, totalValue, shippingAddress), and
  support-ticket-schema.json (7 fields: ticketSubject, reporterName, productName,
  severityLevel, affectedVersion, reproductionSteps, workaroundAvailable).

- src/main/resources/sample-events/seed-documents.jsonl with 3 paired example documents:
  a synthetic vendor invoice (900 words, contains 2 email addresses and a VAT number so S1
  has work to do), a synthetic purchase-order confirmation email (700 words, contains a
  phone number and shipping address), and a synthetic support-ticket email thread (500
  words, contains a user email and a build version identifier).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = automated-action
  (the extracted record is written to the entity without a human gate — extraction is fully
  automated), oversight.human_in_loop = false, failure.failure_modes including
  "missing-required-field", "type-mismatch", "hallucinated-value", "pii-leakage-via-llm",
  "truncated-value"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/extraction-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Structured Extraction Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of extraction job cards; right = selected-job detail with target schema fields,
  sanitized document preview, extracted field table with confidence chips, and coverage badge).
  Browser title exactly: <title>Akka Sample: SchemaExtractor</title>. No subtitle on the
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(jobId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    extract-record.json — 8 ExtractedRecord entries covering the three schema types.
      Each entry has a fields list with one ExtractedField per declared schema field.
      Confidence values vary (HIGH/MEDIUM/LOW/ABSENT) to exercise the coverage badge.
      Plus 2 deliberately MALFORMED entries (one missing a required field; one with a
      fieldName that is not in the submitted schema) — the guardrail blocks both, exercising
      the retry path. The mock selects a malformed entry on the FIRST iteration of every 3rd
      job (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(jobId) helper makes per-job selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ExtractionAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ExtractionTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (extractStep
  60s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the ExtractionJob row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...)
  or .isPresent().
- Lesson 7: ExtractionTasks.java with EXTRACT_RECORD = Task.name(...).description(...)
  .resultConformsTo(ExtractedRecord.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9630 declared explicitly in application.conf's
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
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ExtractionAgent). The
  guardrail (RecordGuardrail.java) is NOT a second agent — it is a deterministic validator.
- The document is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated extractStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The after-llm-response guardrail is wired via the agent's guardrail-configuration
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
