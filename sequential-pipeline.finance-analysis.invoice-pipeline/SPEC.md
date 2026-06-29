# SPEC — invoice-processing

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Invoice Processing.
**One-line pitch:** A user submits an invoice document; one `InvoiceAgent` walks it through three task phases — **EXTRACT** raw line items and header data, **VALIDATE** them against GL accounts and business rules, **POST** a journal entry — with vendor PII redacted at the extract→validate boundary and high-value invoices held at a human-in-the-loop approval gate before the POST phase executes.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a finance-analysis domain. One `InvoiceAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the EXTRACT task's typed output becomes the VALIDATE task's instruction context; the VALIDATE task's typed output becomes the POST task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-posting` sanitizer** (`VendorPiiSanitizer`) runs at the extract→validate boundary before the validated data is written to `InvoiceEntity`. It replaces vendor contact fields (email, phone, individual name) with `[REDACTED]` placeholders. The sanitizer fires unconditionally — every invoice goes through it, regardless of vendor type. The raw PII values are never written to the entity log; the `SanitizationApplied` event records which field names were redacted, not the original values. This is an application-level sanitizer, not an LLM-layer filter.
- A **human-in-the-loop (HITL) approval gate** sits between `validateStep` and `postStep`. After validation, the workflow checks the `ValidatedInvoice.totalAmount`. If the amount exceeds the configured threshold (`invoice.approval-threshold-amount`), the workflow emits `ApprovalRequested`, transitions the entity to `APPROVAL_REQUESTED`, and suspends by entering a workflow timer pause. The workflow resumes only when a reviewer calls `POST /api/invoices/{id}/approve` or `POST /api/invoices/{id}/deny`. A denied invoice transitions to `DENIED` (terminal); an approved invoice continues to `postStep`. Invoices below the threshold skip directly from `VALIDATED` to `POSTING`.

The blueprint shows that a sequential pipeline can combine deterministic data sanitization with an async human decision point — both enforced by workflow-level guards, not by the agent itself.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **vendor** from the seeded vendor dropdown (or uploads a raw text description) and clicks **Submit invoice**. The UI POSTs to `/api/invoices` and receives an `invoiceId`.
2. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `EXTRACTING` — the workflow has started `extractStep` and the agent has been handed the EXTRACT task.
3. Within ~10–20 s the card reaches `EXTRACTED`. The right pane shows the extracted header (vendor name, invoice number, invoice date, currency, total amount) and a line-item table. Vendor PII (contact email, phone) is shown as `[REDACTED]`.
4. Within ~10–20 s more the card reaches `VALIDATED`. The right pane shows the validated line items with assigned GL account codes and a balance check result.
5. If the total amount exceeds the threshold, the card reaches `APPROVAL_REQUESTED` and displays an "Awaiting approval" banner with an Approve / Deny button pair.
6. After the reviewer clicks Approve (or the amount was below threshold), the card reaches `POSTING`, then `POSTED`, then within ~1 s `EVALUATED`. The right pane shows the journal entry (debits and credits) and an eval score chip (1–5).
7. The user can submit another invoice; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InvoiceEndpoint` | `HttpEndpoint` | `/api/invoices/*` — submit, list, get, SSE, approve, deny; serves `/api/metadata/*`. | — | `InvoiceEntity`, `InvoiceView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `InvoiceEntity` | `EventSourcedEntity` | Per-invoice lifecycle: created → extracting → extracted → validating → validated → approval_requested → approved → posting → posted → evaluated (or denied / failed). Source of truth. | `InvoiceEndpoint`, `InvoicePipelineWorkflow` | `InvoiceView` |
| `InvoicePipelineWorkflow` | `Workflow` | One workflow per invoice. Steps: `extractStep` → `validateStep` → `approvalStep` → `postStep` → `evalStep`. The `approvalStep` suspends when amount > threshold; resumes on approve/deny signal. | started by `InvoiceEndpoint` after `CREATED` | `InvoiceAgent`, `InvoiceEntity` |
| `InvoiceAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `InvoiceTasks.java`: `EXTRACT_INVOICE` → `ExtractedInvoice`, `VALIDATE_LINE_ITEMS` → `ValidatedInvoice`, `POST_JOURNAL_ENTRY` → `PostedInvoice`. Each task is registered with the phase-appropriate function tools. | invoked by `InvoicePipelineWorkflow` | returns typed results |
| `ExtractTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `parseHeader(rawText)` and `parseLineItems(rawText)`. Reads from `src/main/resources/sample-data/invoices/*.json` for deterministic offline output. | called from EXTRACT task | returns `InvoiceHeader`, `List<LineItem>` |
| `ValidateTools` | function-tools class | Implements `lookupGlAccount(lineItem)` and `checkBalance(lineItems)`. Pure in-memory lookups against a seeded GL account table. | called from VALIDATE task | returns `GlAccount`, `BalanceResult` |
| `PostTools` | function-tools class | Implements `buildJournalEntry(validatedInvoice)` and `confirmPosting(journalEntry)`. Writes to an in-process ledger stub. | called from POST task | returns `JournalEntry`, `PostingConfirmation` |
| `PhaseGuardrail` | `before-tool-call` guardrail (registered on `InvoiceAgent`) | Reads the in-flight task's declared phase and the current `InvoiceEntity.status`. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `VendorPiiSanitizer` | plain class (no Akka primitive) | Strips vendor contact PII from an `ExtractedInvoice` before it is written to the entity. Records which field names were redacted. | called from `extractStep` after agent returns | returns sanitized `ExtractedInvoice` + `SanitizationReport` |
| `PostingQualityScorer` | plain class (no Akka primitive) | Deterministic on-decision evaluator. Inputs: `PostedInvoice`, `ValidatedInvoice`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `InvoiceView` | `View` | Read model: one row per invoice for the UI. | `InvoiceEntity` events | `InvoiceEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record InvoiceHeader(
    String invoiceNumber,
    String vendorName,
    String vendorEmail,       // PII — redacted after extraction
    String vendorPhone,       // PII — redacted after extraction
    LocalDate invoiceDate,
    String currency,
    BigDecimal totalAmount
) {}

record LineItem(
    String lineId,
    String description,
    int quantity,
    BigDecimal unitPrice,
    BigDecimal lineTotal
) {}

record ExtractedInvoice(
    InvoiceHeader header,
    List<LineItem> lineItems,
    Instant extractedAt
) {}

record SanitizationReport(
    List<String> redactedFields,
    Instant sanitizedAt
) {}

record GlAccount(String accountCode, String accountName, String accountType) {}

record ValidatedLineItem(
    LineItem lineItem,
    GlAccount glAccount,
    boolean balanceOk
) {}

record BalanceResult(boolean balanced, BigDecimal debitTotal, BigDecimal creditTotal) {}

record ValidatedInvoice(
    ExtractedInvoice source,
    List<ValidatedLineItem> lineItems,
    BalanceResult balance,
    BigDecimal totalAmount,
    Instant validatedAt
) {}

record JournalEntry(
    String entryId,
    String invoiceRef,
    List<JournalLine> lines,
    Instant entryDate
) {}

record JournalLine(String accountCode, String accountName, BigDecimal debit, BigDecimal credit) {}

record PostingConfirmation(String confirmationRef, Instant postedAt) {}

record PostedInvoice(
    ValidatedInvoice validated,
    JournalEntry journalEntry,
    PostingConfirmation confirmation,
    Instant postedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record InvoiceRecord(
    String invoiceId,
    Optional<String> rawText,
    Optional<ExtractedInvoice> extracted,
    Optional<SanitizationReport> sanitization,
    Optional<ValidatedInvoice> validated,
    Optional<PostedInvoice> posted,
    Optional<EvalResult> eval,
    InvoiceStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum InvoiceStatus {
    CREATED, EXTRACTING, EXTRACTED, VALIDATING, VALIDATED,
    APPROVAL_REQUESTED, APPROVED, POSTING, POSTED, EVALUATED,
    DENIED, FAILED
}
```

Events on `InvoiceEntity`: `InvoiceCreated`, `ExtractionStarted`, `ExtractionCompleted`, `SanitizationApplied`, `ValidationStarted`, `ValidationCompleted`, `ApprovalRequested`, `ApprovalGranted`, `ApprovalDenied`, `PostingStarted`, `PostingCompleted`, `EvaluationScored`, `PhaseGuardrailRejected`, `InvoiceFailed`.

Every nullable lifecycle field on the `InvoiceRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/invoices` — body `{ rawText }` → `{ invoiceId }`.
- `GET /api/invoices` — list all invoices, newest-first.
- `GET /api/invoices/{id}` — one invoice.
- `GET /api/invoices/sse` — Server-Sent Events; one event per state transition.
- `POST /api/invoices/{id}/approve` — reviewer approves a held invoice; body `{ reviewerNote }`.
- `POST /api/invoices/{id}/deny` — reviewer denies a held invoice; body `{ reviewerNote }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Invoice Processing</title>`.

The App UI tab is a two-column layout: a left rail with the live list of invoices (status pill + vendor name + total amount + age) and a right pane with the selected invoice's detail — header table, sanitization badge, validated line-item table with GL account codes, journal entry lines, eval score chip, and an Approve / Deny button pair when the invoice is in `APPROVAL_REQUESTED` state.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Vendor PII sanitizer**: `VendorPiiSanitizer` runs at the extract→validate boundary, inside `extractStep`, immediately after the agent returns an `ExtractedInvoice`. It iterates over the `InvoiceHeader` fields declared as PII-bearing (`vendorEmail`, `vendorPhone`) and replaces their values with `"[REDACTED]"`. The sanitized `ExtractedInvoice` is what gets written to `InvoiceEntity` via `SanitizationApplied{redactedFields, sanitizedAt}` + `ExtractionCompleted{sanitizedExtract}`. The original field values are never persisted to the entity log, the view, or any downstream component. The `SanitizationApplied` event records only the field names that were redacted, not the original values. The sanitizer is deterministic (no LLM call) and unconditional — it fires on every invoice regardless of the vendor.
- **H1 — High-value posting approval (HITL gate)**: `InvoicePipelineWorkflow.approvalStep` reads `ValidatedInvoice.totalAmount` and compares it against the configured `invoice.approval-threshold-amount` (default: 10 000.00 in the invoice's declared currency). If the amount exceeds the threshold, the step emits `ApprovalRequested` on the entity, transitions to `APPROVAL_REQUESTED`, and pauses via a workflow timer (48-hour window). The workflow resumes on either `POST /api/invoices/{id}/approve` (emits `ApprovalGranted`, continues to `postStep`) or `POST /api/invoices/{id}/deny` (emits `ApprovalDenied`, transitions to `DENIED` terminal). If the 48-hour timer expires without a decision, the workflow emits `InvoiceFailed{reason: "approval-timeout"}`. Invoices below the threshold skip `approvalStep` and proceed directly to `postStep`.

## 9. Agent prompts

- `InvoiceAgent` → `prompts/invoice-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. The agent is explicitly told it will never see raw PII field values — sanitization happens outside the LLM loop.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded invoice `acme-standard-001`; within 60 s the invoice reaches `EVALUATED` with a non-empty header, ≥ 3 validated line items, a balanced journal entry, and an eval score chip on the card.
2. **J2** — User submits the seeded invoice `bigco-highvalue-002` (total above threshold); the workflow pauses at `APPROVAL_REQUESTED`; the reviewer clicks Approve in the UI; the workflow resumes and the invoice reaches `EVALUATED`.
3. **J3** — Every `ExtractedInvoice` written to the entity has `vendorEmail = "[REDACTED]"` and `vendorPhone = "[REDACTED]"`. The `SanitizationApplied` event lists `["vendorEmail", "vendorPhone"]` as redacted fields but contains no original values.
4. **J4** — A posting with unbalanced debits/credits (mock-LLM path) receives eval score ≤ 2 with a rationale naming the imbalance amount; the UI flags the card with a red border.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named invoice-processing demonstrating the sequential-pipeline x finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-finance-analysis-invoice-pipeline. Java package
io.akka.samples.invoiceprocessing. Akka 3.6.0. HTTP port 9183.

Components to wire (exactly):

- 1 AutonomousAgent InvoiceAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/invoice-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  EXTRACT, VALIDATE, and POST tool sets are ALL registered on the agent; phase gating is the
  job of PhaseGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (PhaseGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow InvoicePipelineWorkflow per invoiceId with five steps:
  * extractStep — emits ExtractionStarted on the entity, then calls componentClient
    .forAutonomousAgent(InvoiceAgent.class, "agent-" + invoiceId).runSingleTask(
      TaskDef.instructions("Invoice text:\n" + rawText + "\nPhase: EXTRACT\nParse the header
      and all line items.")
        .metadata("invoiceId", invoiceId)
        .metadata("phase", "EXTRACT")
        .taskType(InvoiceTasks.EXTRACT_INVOICE)
    ). Reads result as ExtractedInvoice. Runs VendorPiiSanitizer on the result. Writes
    InvoiceEntity.recordSanitization(sanitizationReport) then
    InvoiceEntity.recordExtraction(sanitizedExtract). WorkflowSettings.stepTimeout 60s.
  * validateStep — emits ValidationStarted, then runSingleTask with TaskDef.instructions
    (formatValidateContext(sanitizedExtract)) and metadata.phase = "VALIDATE", taskType
    VALIDATE_LINE_ITEMS. Writes InvoiceEntity.recordValidation(validatedInvoice). stepTimeout 60s.
  * approvalStep — reads ValidatedInvoice.totalAmount. If amount <= threshold: returns
    immediately with a pass-through effect (no suspend). If amount > threshold: emits
    ApprovalRequested on the entity and enters a workflow timer pause (48h). On resume: reads
    the ApprovalDecision from the entity — if ApprovalGranted proceeds, if ApprovalDenied
    calls InvoiceEntity.deny() and ends. stepTimeout 172800s (48h).
  * postStep — emits PostingStarted, then runSingleTask with TaskDef.instructions
    (formatPostContext(validatedInvoice)) and metadata.phase = "POST", taskType
    POST_JOURNAL_ENTRY. Writes InvoiceEntity.recordPosting(postedInvoice). stepTimeout 60s.
  * evalStep — runs the deterministic PostingQualityScorer over (postedInvoice,
    validatedInvoice) and writes InvoiceEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(InvoicePipelineWorkflow::error). The error step writes
  InvoiceFailed and ends.

- 1 EventSourcedEntity InvoiceEntity (one per invoiceId). State InvoiceRecord{invoiceId,
  rawText: Optional<String>, extracted: Optional<ExtractedInvoice>,
  sanitization: Optional<SanitizationReport>, validated: Optional<ValidatedInvoice>,
  posted: Optional<PostedInvoice>, eval: Optional<EvalResult>, status: InvoiceStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. InvoiceStatus enum: CREATED,
  EXTRACTING, EXTRACTED, VALIDATING, VALIDATED, APPROVAL_REQUESTED, APPROVED, POSTING,
  POSTED, EVALUATED, DENIED, FAILED. Events: InvoiceCreated{rawText}, ExtractionStarted,
  ExtractionCompleted{extracted}, SanitizationApplied{redactedFields, sanitizedAt},
  ValidationStarted, ValidationCompleted{validated}, ApprovalRequested{totalAmount},
  ApprovalGranted{reviewerNote}, ApprovalDenied{reviewerNote}, PostingStarted,
  PostingCompleted{posted}, EvaluationScored{eval}, PhaseGuardrailRejected{phase, tool,
  reason}, InvoiceFailed{reason}. Commands: create, startExtraction, recordExtraction,
  recordSanitization, startValidation, recordValidation, requestApproval, approve, deny,
  startPosting, recordPosting, recordEvaluation, recordGuardrailRejection, fail,
  getInvoice. emptyState() returns InvoiceRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View InvoiceView with row type InvoiceRow that mirrors InvoiceRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes InvoiceEntity events. ONE
  query getAllInvoices: SELECT * AS invoices FROM invoice_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * InvoiceEndpoint at /api with POST /invoices (body {rawText}; mints invoiceId; calls
    InvoiceEntity.create(rawText); then starts InvoicePipelineWorkflow with id
    "pipeline-" + invoiceId; returns {invoiceId}), GET /invoices (list from
    getAllInvoices, sorted newest-first), GET /invoices/{id} (one row), GET
    /invoices/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /invoices/{id}/approve (body {reviewerNote}; calls InvoiceEntity.approve()),
    POST /invoices/{id}/deny (body {reviewerNote}; calls InvoiceEntity.deny()), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- InvoiceTasks.java declaring three Task<R> constants:
    EXTRACT_INVOICE = Task.name("Extract invoice").description("Parse the invoice header and
      all line items from the raw text").resultConformsTo(ExtractedInvoice.class);
    VALIDATE_LINE_ITEMS = Task.name("Validate line items").description("Look up the GL account
      for each line item and check overall balance").resultConformsTo(ValidatedInvoice.class);
    POST_JOURNAL_ENTRY = Task.name("Post journal entry").description("Build and confirm a
      balanced journal entry from the validated invoice").resultConformsTo(PostedInvoice.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {EXTRACT, VALIDATE, POST}. Each function-tool method is annotated with
  the constant phase. PhaseGuardrail reads it from a parallel registry built at startup.

- ExtractTools.java — @FunctionTool parseHeader(String rawText) -> InvoiceHeader reading from
  src/main/resources/sample-data/invoices/*.json keyed by a slug derived from the first line
  of rawText; @FunctionTool parseLineItems(String rawText) -> List<LineItem> reading from the
  matching invoice file's lineItems array.

- ValidateTools.java — @FunctionTool lookupGlAccount(LineItem) -> GlAccount (in-memory lookup
  against src/main/resources/sample-data/gl-accounts.json); @FunctionTool
  checkBalance(List<ValidatedLineItem>) -> BalanceResult (deterministic debit/credit sum).

- PostTools.java — @FunctionTool buildJournalEntry(ValidatedInvoice) -> JournalEntry (one
  JournalLine per ValidatedLineItem, debit to expense account, credit to AP account);
  @FunctionTool confirmPosting(JournalEntry) -> PostingConfirmation (mints confirmationRef
  as "conf-" + sha1(entryId).substring(0,8), records to in-process ledger stub Map).

- VendorPiiSanitizer.java — pure deterministic logic (no LLM, no Akka primitive). Input:
  ExtractedInvoice. Fields to redact: InvoiceHeader.vendorEmail, InvoiceHeader.vendorPhone.
  Returns: sanitized ExtractedInvoice (header rebuilt with "[REDACTED]" in those fields) +
  SanitizationReport{redactedFields: ["vendorEmail","vendorPhone"], sanitizedAt}. NEVER logs
  or returns the original field values.

- PhaseGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's phase attribute, looks up the InvoiceEntity status by invoiceId (carried in the
  TaskDef metadata), applies the accept matrix:
    EXTRACT tools require status ∈ {CREATED, EXTRACTING};
    VALIDATE tools require status ∈ {EXTRACTED, VALIDATING} AND extracted.isPresent();
    POST tools require status ∈ {APPROVED, POSTING} OR (status ∈ {VALIDATED, POSTING} AND
      totalAmount <= threshold);
  On reject ALSO calls InvoiceEntity.recordGuardrailRejection(phase, tool, reason) so the
  rejection is visible in the UI and in the audit log.

- PostingQualityScorer.java — pure deterministic logic (no LLM). Inputs: PostedInvoice,
  ValidatedInvoice. Outputs: EvalResult. Four checks, one point per check satisfied, base 1:
  (1) GL coverage — every ValidatedLineItem has a non-null GlAccount.accountCode in the
    matching JournalLine;
  (2) balance — JournalEntry debit total equals credit total;
  (3) line parity — journalEntry.lines.size() >= validatedInvoice.lineItems.size();
  (4) confirmation present — PostingConfirmation.confirmationRef is non-blank.
  Score 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9183 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Also: invoice.approval-threshold-amount = 10000.00

- src/main/resources/sample-data/invoices/*.json — three files:
    acme-standard-001.json — standard invoice, total 3 200.00 USD, 3 line items.
    bigco-highvalue-002.json — invoice, total 47 500.00 USD, 5 line items (triggers HITL).
    startup-simple-003.json — minimal invoice, total 850.00 USD, 2 line items.
  Each file carries header + lineItems arrays matching the domain records.

- src/main/resources/sample-data/gl-accounts.json — 10 GL account entries covering AP,
  office supplies, software, professional services, and travel expense categories.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, H1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for finance-analysis domain.

- prompts/invoice-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Invoice Processing", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of invoice cards; right = selected-invoice detail with header table, sanitization
  badge, validated line-item table, journal entry, eval-score chip, Approve/Deny buttons when
  in APPROVAL_REQUESTED). Browser title exactly: <title>Akka Sample: Invoice Processing</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider returning deterministic
        typed-correct outputs per Task (collect, validate, post). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; gone when
        the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry by
  MockModelProvider.seedFor(invoiceId), and deserialises into the task's typed return. Each
  entry carries a "tool_calls" array the mock replays before returning the typed result.
- Per-task mock-response shapes:
    extract-invoice.json — 3 ExtractedInvoice entries, one per seeded invoice file.
      Each entry's tool_calls: [parseHeader, parseLineItems]. Plus 1 deliberately
      PHASE-VIOLATING entry whose tool_calls starts with buildJournalEntry (a POST-phase tool
      during EXTRACT) — guardrail rejects it, mock falls through to normal extract. Selected
      on the first iteration of every 3rd invoice (modulo seed) so J3 is reproducible.
    validate-line-items.json — 3 ValidatedInvoice entries paired with the extract entries.
      tool_calls: [lookupGlAccount × N, checkBalance].
    post-journal-entry.json — 3 PostedInvoice entries. tool_calls: [buildJournalEntry,
      confirmPosting]. Plus 1 UNBALANCED entry where debit total != credit total — evalStep
      scores it ≤ 2; J4 verifies this.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. InvoiceAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion InvoiceTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (extractStep
  60s, validateStep 60s, postStep 60s, evalStep 5s, approvalStep 172800s, error 5s).
- Lesson 6: every nullable lifecycle field on the InvoiceRecord row record is Optional<T>.
- Lesson 7: InvoiceTasks.java with EXTRACT_INVOICE, VALIDATE_LINE_ITEMS, POST_JOURNAL_ENTRY
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9183 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND themeVariables
  block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: exactly ONE AutonomousAgent (InvoiceAgent). VendorPiiSanitizer
  and PostingQualityScorer are rule-based and do NOT make LLM calls.
- The sequential-pipeline invariant: all phase tool sets registered on the agent; PhaseGuardrail
  is the runtime gate. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: extractStep writes ExtractedInvoice onto
  the entity, validateStep reads it and builds the VALIDATE task's instruction context,
  postStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
