# SPEC — bank-support-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BankSupportAgent.
**One-line pitch:** A customer submits a banking enquiry; one AI agent looks up their account data through a tool call and returns a structured `SupportResponse` — an `answer` text, a 1–10 `riskScore`, and a `blockCard` flag that signals whether card-blocking is recommended.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `BankSupportAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its context and audit its output. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every `AccountLookupTool` invocation and blocks any call that sets `blockCard=true` unless the enquiry's `riskScore` has already been recorded above threshold (score ≥ 7) — preventing the agent from triggering a destructive account action on a routine enquiry.
- A **before-agent-response guardrail** validates the candidate `SupportResponse` on every turn: well-formed JSON, `riskScore` in `[1, 10]`, `blockCard` consistent with the risk score, and `answer` non-empty. A response that fails any check is rejected inside the agent loop for a retry.
- A **PII sanitizer** runs inside a Consumer that subscribes to `ResponseRecorded` events; it redacts account numbers, sort codes, full names, email addresses, and phone numbers before writing the sanitized response to the audit log view. The raw response remains on the entity for compliance but never appears in the live UI read model.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **customer account** from a dropdown (or enters a free-form `customerId`) and types an **enquiry** into the text area (or selects one of three seeded examples: a balance query, a disputed transaction, a lost-card report).
2. The user clicks **Submit enquiry**. The UI POSTs to `/api/enquiries` and receives an `enquiryId`.
3. The card appears in the live list in `SUBMITTED` state.
4. Within ~1 s the workflow's `lookupStep` completes. The card transitions to `ACCOUNT_LOADED` — the account summary is visible in the right pane (masked account number, available balance, account type).
5. Within ~10–30 s the `respondStep` completes. The card transitions to `RESPONDING` then `RESPONSE_RECORDED`. The response appears: a risk score badge (1–10), a `blockCard` indicator, and the `answer` paragraph.
6. Within ~1 s the `sanitizeStep` finishes. The card reaches `LOGGED` and the sanitized response label appears — confirming PII has been stripped from the audit log.
7. The user can submit another enquiry; the live list keeps the full history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EnquiryEndpoint` | `HttpEndpoint` | `/api/enquiries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `EnquiryEntity`, `EnquiryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `EnquiryEntity` | `EventSourcedEntity` | Per-enquiry lifecycle: submitted → account_loaded → responding → response_recorded → logged. Source of truth. | `EnquiryEndpoint`, `EnquiryWorkflow` | `EnquiryView` |
| `EnquiryWorkflow` | `Workflow` | One workflow per enquiry. Steps: `lookupStep` → `respondStep` → `sanitizeStep`. | started by `EnquiryEndpoint` after submit | `BankSupportAgent`, `EnquiryEntity` |
| `BankSupportAgent` | `AutonomousAgent` | The one decision-making LLM. Receives customer context in task instructions and calls `AccountLookupTool` for live data; returns `SupportResponse`. | invoked by `EnquiryWorkflow` | returns response |
| `AccountLookupTool` | tool call (in-process) | Returns `AccountSummary` for a `customerId`; also accepts `blockCard=true` to flag card-block requests. | invoked by `BankSupportAgent` | — |
| `ToolCallGuardrail` | supporting class | Intercepts `AccountLookupTool` calls; vetoes `blockCard=true` unless risk threshold is met. | before-tool-call hook on `BankSupportAgent` | — |
| `ResponseGuardrail` | supporting class | Validates `SupportResponse` structure, score range, and blockCard consistency. | before-agent-response hook on `BankSupportAgent` | — |
| `ResponseSanitizer` | `Consumer` | Subscribes to `ResponseRecorded` events; redacts PII; calls `EnquiryEntity.attachSanitizedLog`. | `EnquiryEntity` events | `EnquiryEntity` |
| `EnquiryView` | `View` | Read model: one row per enquiry for the UI. Shows sanitized response only. | `EnquiryEntity` events | `EnquiryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CustomerEnquiry(
    String enquiryId,
    String customerId,
    String enquiryText,
    EnquiryCategory category,
    String submittedBy,
    Instant submittedAt
) {}
enum EnquiryCategory { BALANCE_QUERY, DISPUTED_TRANSACTION, LOST_CARD, GENERAL }

record AccountSummary(
    String customerId,
    String maskedAccountNumber,   // e.g. "****4821"
    String accountType,           // "CHECKING" | "SAVINGS" | "CREDIT"
    BigDecimal availableBalance,
    String currencyCode,
    boolean cardActive
) {}

record SupportResponse(
    String answer,
    int riskScore,                // 1..10
    boolean blockCard,
    ResponseTone tone,
    Instant decidedAt
) {}
enum ResponseTone { REASSURING, NEUTRAL, URGENT }

record SanitizedLog(
    String redactedAnswer,
    List<String> piiCategoriesRedacted
) {}

record Enquiry(
    String enquiryId,
    Optional<CustomerEnquiry> enquiry,
    Optional<AccountSummary> account,
    Optional<SupportResponse> response,
    Optional<SanitizedLog> sanitizedLog,
    EnquiryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EnquiryStatus {
    SUBMITTED, ACCOUNT_LOADED, RESPONDING, RESPONSE_RECORDED, LOGGED, FAILED
}
```

Events on `EnquiryEntity`: `EnquirySubmitted`, `AccountLoaded`, `RespondingStarted`, `ResponseRecorded`, `LogSanitized`, `EnquiryFailed`.

Every nullable lifecycle field on the `Enquiry` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/enquiries` — body `{ customerId, enquiryText, category, submittedBy }` → `{ enquiryId }`.
- `GET /api/enquiries` — list all enquiries, newest-first.
- `GET /api/enquiries/{id}` — one enquiry.
- `GET /api/enquiries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Bank Support Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted enquiries (status pill + risk badge + age) and a right pane with the selected enquiry's detail — account summary card, enquiry text, support response answer, risk score badge, blockCard indicator, and sanitized-log confirmation chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`ToolCallGuardrail`, bound to the `before-tool-call` hook on `BankSupportAgent`): intercepts every `AccountLookupTool` invocation. If the tool parameters include `blockCard=true` and the current `riskScore` on the running enquiry context is below 7, the call is blocked with a structured error explaining the pre-condition, forcing the agent to re-plan. If `riskScore ≥ 7` the call is allowed through. All non-blocking lookups (balance, transaction history) pass unconditionally.
- **G2 — before-agent-response guardrail** (`ResponseGuardrail`, bound to the `before-agent-response` hook): validates every candidate `SupportResponse` — (1) parses into the record type, (2) asserts `riskScore` is in `[1, 10]`, (3) asserts if `blockCard == true` then `riskScore ≥ 7`, (4) asserts `answer` is non-empty. On any failure returns a structured rejection so the agent loop retries within its 3-iteration budget.
- **S1 — PII sanitizer** (`ResponseSanitizer` Consumer, `pii` flavor): runs after `ResponseRecorded` lands. Redacts account numbers, sort codes, full names, email addresses, and phone numbers from the `answer` text before writing the `SanitizedLog` to the entity. The view and UI display only the sanitized form; the raw response remains on the entity for compliance.

## 9. Agent prompts

- `BankSupportAgent` → `prompts/bank-support-agent.md`. The single decision-making LLM. System prompt instructs it to look up the customer account, assess the enquiry, and return a `SupportResponse` with a calibrated risk score and a card-blocking recommendation only when genuinely warranted.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a balance query for a seeded account; within 30 s the card reaches `LOGGED` with a `SupportResponse` carrying a risk score of 1–3 and `blockCard = false`.
2. **J2** — Agent calls `AccountLookupTool` with `blockCard=true` on a low-risk enquiry (risk score context < 7) → `ToolCallGuardrail` vetoes it → the agent re-plans and submits the response without the card-block.
3. **J3** — The agent's first response carries `riskScore = 15` → `ResponseGuardrail` rejects it → the agent retries → a valid response with `riskScore` in `[1, 10]` reaches the entity.
4. **J4** — A response containing `ACCT-****4821` and the customer's full name is recorded → the sanitized log in the UI shows `[REDACTED-ACCOUNT]` and `[REDACTED-NAME]`; the entity retains the raw form.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named bank-support-agent demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-bank-support-agent. Java package io.akka.samples.banksupportagent.
Akka 3.6.0. HTTP port 9303.

Components to wire (exactly):

- 1 AutonomousAgent BankSupportAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/bank-support-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_ENQUIRY).maxIterationsPerTask(3)). The task receives
  the customer context (customerId, enquiryText, category, maskedAccountNumber) as its
  instruction text. The agent calls AccountLookupTool (in-process tool) to fetch account data.
  Output: SupportResponse{answer: String, riskScore: int, blockCard: boolean, tone:
  ResponseTone, decidedAt: Instant}. The agent is configured with two guardrails — a
  before-tool-call guardrail (ToolCallGuardrail) and a before-agent-response guardrail
  (ResponseGuardrail) — registered via the agent's guardrail-configuration block.

- 1 Workflow EnquiryWorkflow per enquiryId with three steps:
  * lookupStep — calls AccountLookupTool.lookup(customerId) synchronously (in-process),
    emits AccountLoaded on EnquiryEntity with the AccountSummary. WorkflowSettings.stepTimeout
    10s.
  * respondStep — emits RespondingStarted, then calls componentClient.forAutonomousAgent(
    BankSupportAgent.class, "support-" + enquiryId).runSingleTask(
      TaskDef.instructions(formatContext(enquiry, account))
    ) to fetch a taskId, then forTask(taskId).result(HANDLE_ENQUIRY). On success calls
    EnquiryEntity.recordResponse(response). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(EnquiryWorkflow::error).
  * sanitizeStep — fires synchronously; calls ResponseSanitizer.sanitize(response.answer)
    in-process, builds SanitizedLog, calls EnquiryEntity.attachSanitizedLog. WorkflowSettings
    .stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity EnquiryEntity (one per enquiryId). State Enquiry{enquiryId: String,
  enquiry: Optional<CustomerEnquiry>, account: Optional<AccountSummary>, response:
  Optional<SupportResponse>, sanitizedLog: Optional<SanitizedLog>, status: EnquiryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. EnquiryStatus enum: SUBMITTED,
  ACCOUNT_LOADED, RESPONDING, RESPONSE_RECORDED, LOGGED, FAILED. Events: EnquirySubmitted{
  enquiry}, AccountLoaded{account}, RespondingStarted{}, ResponseRecorded{response},
  LogSanitized{sanitizedLog}, EnquiryFailed{reason}. Commands: submit, loadAccount,
  startResponding, recordResponse, attachSanitizedLog, fail, getEnquiry. emptyState() returns
  Enquiry.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside event-appliers.

- 1 Consumer ResponseSanitizer subscribed to EnquiryEntity events; on ResponseRecorded runs
  a regex+heuristic redaction pipeline (account numbers matching ACCT-\d+, sort codes
  \d{2}-\d{2}-\d{2}, full names heuristic, email addresses, phone numbers) over response
  .answer, computes the list of categories found, builds SanitizedLog, then calls
  EnquiryEntity.attachSanitizedLog(sanitizedLog).

- 1 View EnquiryView with row type EnquiryRow (mirrors Enquiry minus the raw response.answer
  — the audit entity keeps the raw; the view holds the sanitized form for the UI). Table
  updater consumes EnquiryEntity events. ONE query getAllEnquiries: SELECT * AS enquiries FROM
  enquiry_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * EnquiryEndpoint at /api with POST /enquiries (body {customerId, enquiryText, category,
    submittedBy}; mints enquiryId; calls EnquiryEntity.submit; starts EnquiryWorkflow;
    returns {enquiryId}), GET /enquiries (list from getAllEnquiries, sorted newest-first),
    GET /enquiries/{id} (one row), GET /enquiries/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- EnquiryTasks.java declaring one Task<R> constant: HANDLE_ENQUIRY = Task.name("Handle
  enquiry").description("Review the customer's enquiry and account data, then produce a
  SupportResponse with a risk score and card-blocking recommendation").resultConformsTo(
  SupportResponse.class). DO NOT skip this — the AutonomousAgent requires its companion Tasks
  class (Lesson 7).

- Domain records CustomerEnquiry, EnquiryCategory, AccountSummary, SupportResponse,
  ResponseTone, SanitizedLog, Enquiry, EnquiryStatus.

- AccountLookupTool.java — in-process implementation. lookup(customerId) returns an
  AccountSummary loaded from src/main/resources/sample-events/accounts.jsonl (keyed by
  customerId). blockCardRequest(customerId) records a card-block request and returns an
  updated AccountSummary with cardActive=false. The tool is registered on BankSupportAgent
  via .tool(AccountLookupTool.class) in the AgentDefinition.

- ToolCallGuardrail.java implementing the before-tool-call hook on BankSupportAgent. Checks
  that any AccountLookupTool invocation carrying blockCard=true has a preceding riskScore ≥ 7
  in the current task context. On failure returns Guardrail.block(<structured-error>) naming
  the pre-condition that was unmet.

- ResponseGuardrail.java implementing the before-agent-response hook. Reads the candidate
  SupportResponse, runs the four checks listed in eval-matrix.yaml G2, and either passes the
  response through or returns Guardrail.reject(<structured-error>) to force the agent loop to
  retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9303 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. BankSupportAgent.definition() binds the
  configured provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/accounts.jsonl with 5 seeded customer account fixtures:
  2 checking accounts (one with a flagged recent transaction), 2 savings accounts, and 1
  credit account with a past-due balance. Each fixture contains 2–3 plausible PII strings
  (name, email, phone) in the response answer field so S1 has work to do.

- src/main/resources/sample-events/enquiries.jsonl with 3 seeded enquiry examples:
  a balance query (low risk, blockCard=false expected), a disputed-transaction report (medium
  risk), and a lost-card report (high risk, blockCard=true expected). Each includes a
  customerId matching a fixture in accounts.jsonl.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, G2, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — cx-support
  is the use case, not the regulatory anchor (deployer fills in jurisdictions).

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  decisions.authority_level = recommend-only (the agent recommends card-blocking; a human or
  downstream system acts on it), oversight.human_in_loop = true,
  failure.failure_modes including "false-card-block", "missed-fraud-signal",
  "pii-leakage-via-log", "risk-score-miscalibration"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/bank-support-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Bank Support Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of enquiry cards; right = selected-enquiry detail with account summary, enquiry
  text, support response, risk badge, blockCard indicator, and sanitized-log chip). Browser
  title exactly: <title>Akka Sample: Bank Support Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on the Task<R> id. Each branch
  reads src/main/resources/mock-responses/<task-id>.json, picks one entry deterministically
  per call (seedFor(enquiryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    handle-enquiry.json — 8 SupportResponse entries covering the full ResponseTone range and
    riskScore spread (1–10). Entries include realistic answer paragraphs, varied blockCard
    values, and entries with riskScore consistent with blockCard=true (score ≥ 7). Plus 2
    deliberately MALFORMED entries (one with riskScore=15; one with blockCard=true and
    riskScore=2) — the guardrail blocks both, exercising the retry path. The mock selects a
    malformed entry on the FIRST iteration of every 3rd enquiry (modulo seed) so J3 is
    reproducible.
- A MockModelProvider.seedFor(enquiryId) helper makes per-enquiry selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BankSupportAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion EnquiryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (lookupStep 10s, respondStep 60s,
  sanitizeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Enquiry row record is Optional<T>.
- Lesson 7: EnquiryTasks.java with HANDLE_ENQUIRY = Task.name(...).description(...)
  .resultConformsTo(SupportResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9303 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the mermaid.initialize
  themeVariables block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BankSupportAgent).
  ResponseSanitizer is a Consumer — not an LLM call — keeping the pattern honest.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external wrapper. It fires before the tool executes, so a vetoed blockCard call
  never reaches AccountLookupTool.blockCardRequest.
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
