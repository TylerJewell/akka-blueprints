# SPEC — ambient-expense-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AmbientExpenseAgent.
**One-line pitch:** A user submits raw receipt text or a voice transcript; one AI agent extracts and categorizes expense line items (passed as a task attachment, never as inline prompt text) and returns a structured `ExpenseReport` ready for submission to the expense system.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `ExpenseCaptureAgent` (AutonomousAgent) carries the entire extraction and categorization decision; the surrounding components only prepare its input and govern its output actions. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw receipt submission and the agent call — so the model never sees card numbers, employee names, or street addresses in their unredacted form.
- A **before-tool-call guardrail** validates every tool call the agent would make to submit a line item to the expense system. Checks include: amount within policy limit, category in the approved taxonomy, currency code valid, merchant name present. A rejected tool call forces the agent to reclassify or flag the item rather than submitting blindly.

The blueprint shows that governance in a single-agent system does not require a second LLM. Two independent checks sit on either side of the one decision-making model call, and neither of them makes an LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes raw receipt text or a voice-transcript into the **Receipt** textarea (or loads one of three seeded examples — a restaurant receipt, a hotel folio, a mixed-vendor travel itinerary).
2. The user fills in a **Submitter ID** and an optional **Trip or project code**.
3. The user clicks **Submit receipt**. The UI POSTs to `/api/expenses` and receives an `expenseId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted receipt is visible in the card detail, with a small list of PII categories the sanitizer found (e.g., `card-number`, `person-name`, `address`).
5. Within ~10–30 s, the workflow's `captureStep` completes. The card transitions to `CAPTURING` then `REPORT_READY`. The expense report appears: a line-item table (description, amount, currency, category, merchant, date), a total amount, and a status column showing `APPROVED`, `FLAGGED`, or `BLOCKED` per line.
6. Any line item blocked by the before-tool-call guardrail shows a `BLOCKED` badge and the guardrail's rejection reason, giving the submitter clear feedback on why that line item was held.
7. The user can submit another receipt; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ExpenseEndpoint` | `HttpEndpoint` | `/api/expenses/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ExpenseEntity`, `ExpenseView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ExpenseEntity` | `EventSourcedEntity` | Per-submission lifecycle: submitted → sanitized → capturing → report-ready → submitted-to-system. Source of truth. | `ExpenseEndpoint`, `ReceiptSanitizer`, `ExpenseWorkflow` | `ExpenseView` |
| `ReceiptSanitizer` | `Consumer` | Subscribes to `ReceiptSubmitted` events; redacts PII; calls `ExpenseEntity.attachSanitized`. | `ExpenseEntity` events | `ExpenseEntity` |
| `ExpenseWorkflow` | `Workflow` | One workflow per submission. Steps: `awaitSanitizedStep` → `captureStep` → `submitStep`. | started by `ReceiptSanitizer` once sanitized event lands | `ExpenseCaptureAgent`, `ExpenseEntity` |
| `ExpenseCaptureAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the expense-category taxonomy in the task definition and the sanitized receipt as a task attachment; returns `ExpenseReport`. | invoked by `ExpenseWorkflow` | returns report |
| `SubmissionGuardrail` | before-tool-call hook | Validates every tool call that would write a line item to the expense system. | registered on `ExpenseCaptureAgent` | blocks or passes each tool call |
| `ExpenseView` | `View` | Read model: one row per submission for the UI. | `ExpenseEntity` events | `ExpenseEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ExpenseCategory(String categoryId, String displayName, String glCode) {}

record SubmitReceiptRequest(
    String expenseId,
    String rawReceiptText,
    String submittedBy,
    Optional<String> tripCode,
    Instant submittedAt
) {}

record SanitizedReceipt(
    String redactedText,
    List<String> piiCategoriesFound
) {}

record ExpenseLineItem(
    String lineItemId,
    String description,
    BigDecimal amount,
    String currencyCode,
    String categoryId,
    String merchantName,
    LocalDate expenseDate,
    LineItemStatus lineStatus,
    Optional<String> blockReason
) {}
enum LineItemStatus { APPROVED, FLAGGED, BLOCKED }

record ExpenseReport(
    String expenseId,
    List<ExpenseLineItem> lineItems,
    BigDecimal totalAmount,
    String currencyCode,
    ReportStatus reportStatus,
    Instant capturedAt
) {}
enum ReportStatus { FULLY_APPROVED, PARTIALLY_BLOCKED, FULLY_BLOCKED }

record ExpenseSubmission(
    String expenseId,
    Optional<SubmitReceiptRequest> request,
    Optional<SanitizedReceipt> sanitized,
    Optional<ExpenseReport> report,
    SubmissionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SubmissionStatus {
    SUBMITTED, SANITIZED, CAPTURING, REPORT_READY, SUBMITTED_TO_SYSTEM, FAILED
}
```

Events on `ExpenseEntity`: `ReceiptSubmitted`, `ReceiptSanitized`, `CaptureStarted`, `ReportReady`, `SystemSubmitted`, `SubmissionFailed`.

Every nullable lifecycle field on the `ExpenseSubmission` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/expenses` — body `{ rawReceiptText, submittedBy, tripCode? }` → `{ expenseId }`.
- `GET /api/expenses` — list all submissions, newest-first.
- `GET /api/expenses/{id}` — one submission.
- `GET /api/expenses/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Ambient Expense Capture</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted receipts (status pill + report-status badge + age) and a right pane with the selected submission's detail — sanitized receipt preview, line-item table with per-item status, total amount, and any guardrail block reasons.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `ReceiptSanitizer` Consumer): redacts payment card numbers (PAN, partial card tokens), employee names, street addresses, phone numbers, and account-like tokens from the raw receipt text before any LLM call. Records which categories were found.
- **G1 — before-tool-call guardrail**: runs before every tool call `ExpenseCaptureAgent` would make to submit a line item. Checks: amount does not exceed configured per-category policy limit, `categoryId` is present in the approved taxonomy, `currencyCode` is a valid ISO 4217 code, `merchantName` is non-empty. On failure, returns a structured rejection specifying which check failed; the agent loop either corrects the item or marks it `BLOCKED` and proceeds with remaining items.

## 9. Agent prompts

- `ExpenseCaptureAgent` → `prompts/expense-capture-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached receipt text, extract every discernible expense line item, categorize each against the provided taxonomy, and attempt to submit each via tool call — deferring to the guardrail's feedback for any item that fails validation.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the restaurant seed receipt; within 30 s the report appears with one line item per identifiable charge and a `FULLY_APPROVED` report status.
2. **J2** — User submits the hotel folio seed; the folio contains a minibar charge that exceeds the per-item policy limit; the before-tool-call guardrail blocks that line item; the report shows `PARTIALLY_BLOCKED` with a clear block reason; all other items are `APPROVED`.
3. **J3** — A receipt containing a full 16-digit card number and a street address is submitted; the LLM call log shows only `[REDACTED-CARD]` and `[REDACTED-ADDRESS]`; the entity's `request.rawReceiptText` retains the raw text for audit.
4. **J4** — User submits a voice transcript with ambiguous phrasing ("fifty-three bucks, coffee and maybe a muffin"); the agent extracts what it can, marks uncertain items `FLAGGED`, and returns a partial report rather than refusing the task.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ambient-expense-agent demonstrating the single-agent × finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-ambient-expense-capture. Java package
io.akka.samples.ambientexpenseagent. Akka 3.6.0. HTTP port 9483.

Components to wire (exactly):

- 1 AutonomousAgent ExpenseCaptureAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/expense-capture-agent.md>) and
  .capability(TaskAcceptance.of(CAPTURE_EXPENSE).maxIterationsPerTask(4)). The task receives
  the expense-category taxonomy as its instruction text and the sanitized receipt as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is
  the canonical call). Output: ExpenseReport{expenseId: String, lineItems:
  List<ExpenseLineItem>, totalAmount: BigDecimal, currencyCode: String, reportStatus:
  ReportStatus, capturedAt: Instant}. The agent is configured with a before-tool-call
  guardrail (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop receives a structured error and may correct
  the item or mark it BLOCKED within its 4-iteration budget.

- 1 Workflow ExpenseWorkflow per expenseId with three steps:
  * awaitSanitizedStep — polls ExpenseEntity.getSubmission every 1s; on
    submission.sanitized().isPresent() advances to captureStep. WorkflowSettings.stepTimeout
    15s (sanitizer is in-process and fast).
  * captureStep — emits CaptureStarted, then calls componentClient.forAutonomousAgent(
    ExpenseCaptureAgent.class, "capture-" + expenseId).runSingleTask(
      TaskDef.instructions(formatTaxonomy(categories))
        .attachment("receipt.txt", submission.sanitized.redactedText.getBytes())
    ) — returns a taskId, then forTask(taskId).result(CAPTURE_EXPENSE) to fetch the report.
    On success calls ExpenseEntity.recordReport(report). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(ExpenseWorkflow::error).
  * submitStep — calls ExpenseEntity.markSystemSubmitted. WorkflowSettings.stepTimeout 10s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ExpenseEntity (one per expenseId). State ExpenseSubmission{expenseId:
  String, request: Optional<SubmitReceiptRequest>, sanitized: Optional<SanitizedReceipt>,
  report: Optional<ExpenseReport>, status: SubmissionStatus, createdAt: Instant, finishedAt:
  Optional<Instant>}. SubmissionStatus enum: SUBMITTED, SANITIZED, CAPTURING, REPORT_READY,
  SUBMITTED_TO_SYSTEM, FAILED. Events: ReceiptSubmitted{request}, ReceiptSanitized{sanitized},
  CaptureStarted{}, ReportReady{report}, SystemSubmitted{}, SubmissionFailed{reason}. Commands:
  submit, attachSanitized, markCapturing, recordReport, markSystemSubmitted, fail,
  getSubmission. emptyState() returns ExpenseSubmission.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer ReceiptSanitizer subscribed to ExpenseEntity events; on ReceiptSubmitted runs a
  regex+heuristic redaction pipeline (payment-card numbers, person names, street addresses,
  phone numbers, account-id-like tokens) over rawReceiptText, computes the list of categories
  found, builds SanitizedReceipt, then calls ExpenseEntity.attachSanitized(sanitized). After
  attachSanitized lands, the same Consumer starts an ExpenseWorkflow with id = "expense-" +
  expenseId.

- 1 View ExpenseView with row type ExpenseRow (mirrors ExpenseSubmission minus
  request.rawReceiptText — the audit log keeps the raw; the view holds the sanitized form for
  the UI). Table updater consumes ExpenseEntity events. ONE query getAllSubmissions: SELECT *
  AS submissions FROM expense_view. No WHERE status filter — Akka cannot auto-index enum
  columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ExpenseEndpoint at /api with POST /expenses (body {rawReceiptText, submittedBy,
    tripCode?}; mints expenseId; calls ExpenseEntity.submit; returns {expenseId}), GET
    /expenses (list from getAllSubmissions, sorted newest-first), GET /expenses/{id} (one
    row), GET /expenses/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ExpenseTasks.java declaring one Task<R> constant: CAPTURE_EXPENSE = Task.name("Capture
  expense").description("Read the attached receipt and produce an ExpenseReport with one
  ExpenseLineItem per discernible charge, categorized against the provided taxonomy")
  .resultConformsTo(ExpenseReport.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ExpenseCategory, SubmitReceiptRequest, SanitizedReceipt, ExpenseLineItem,
  LineItemStatus, ExpenseReport, ReportStatus, ExpenseSubmission, SubmissionStatus.

- SubmissionGuardrail.java implementing the before-tool-call hook. Reads the candidate tool
  call (a line-item submission), runs the four checks listed in eval-matrix.yaml G1 (amount
  within limit, category in taxonomy, currency valid, merchant non-empty), and either passes
  the tool call through or returns Guardrail.reject(<structured-error>) with the failed
  check named.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9483 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ExpenseCaptureAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/expense-categories.jsonl with the approved expense
  taxonomy: 8 categories (meals-and-entertainment, accommodation, ground-transport,
  air-travel, conference-and-training, software-and-subscriptions, office-supplies,
  other). Each entry has categoryId, displayName, glCode, and maxAmountUsd.

- src/main/resources/sample-events/seed-receipts.jsonl with 3 seed receipts: a restaurant
  receipt ($47 dinner, 2 items), a hotel folio (3 nights + minibar charge that exceeds
  policy), and a mixed travel itinerary (flight + taxi + coffee). Each contains 1–2 PII
  strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous
  (for approved items; blocked items require human resolution), oversight.human_in_loop =
  true (for BLOCKED items), failure.failure_modes including "missed-line-item",
  "wrong-category", "pii-leakage-via-llm", "amount-extraction-error",
  "policy-limit-bypass"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/expense-capture-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Ambient Expense Capture",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of submission cards; right = selected-submission detail with sanitized receipt
  preview, line-item table showing per-item status, total amount, and block reasons).
  Browser title exactly: <title>Akka Sample: Ambient Expense Capture</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; gone when
        the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id.
  Each branch reads src/main/resources/mock-responses/<task-id>.json, picks one entry
  pseudo-randomly per call (seedFor(expenseId)), and deserialises into the task's typed
  return.
- Per-task mock-response shapes for THIS blueprint:
    capture-expense.json — 6 ExpenseReport entries covering the three ReportStatus values.
      Each entry has a lineItems array matching the typical charge pattern of the seeded
      receipts. Amounts, categories, and merchants vary. Plus 1 entry where a line item's
      categoryId is not in the taxonomy (guardrail blocks it) and 1 entry where a line
      item's amount exceeds the per-category limit (guardrail blocks that item), so J2 is
      exercised without a real LLM.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ExpenseCaptureAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ExpenseTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (captureStep
  60s, awaitSanitizedStep 15s, submitStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the ExpenseSubmission row record is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: ExpenseTasks.java with CAPTURE_EXPENSE = Task.name(...).description(...)
  .resultConformsTo(ExpenseReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9483 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ExpenseCaptureAgent).
  The submission guardrail (SubmissionGuardrail.java) is a before-tool-call hook and does
  NOT make an LLM call — keeping the pattern's "one agent" promise honest.
- The receipt is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated captureStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check after the agent returns.
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
