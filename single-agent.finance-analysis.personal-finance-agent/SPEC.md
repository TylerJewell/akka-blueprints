# SPEC — personal-finance-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** PersonalFinanceAssistant.
**One-line pitch:** A user asks a natural-language question about their accounts and spending; one AI agent calls a set of bank and finance tools to answer it, with PII stripped from all transaction data before the model sees them and every write-intent validated by a guardrail before the tool fires.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `FinanceAssistantAgent` (AutonomousAgent) carries every decision — which tools to call, how to interpret the results, how to compose the answer. The surrounding components prepare safe inputs, protect write paths, and record the full interaction audit trail. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw query submission and the agent call — account numbers are tokenised, card PAN fragments are masked, full-name strings are replaced with initials — so the model never sees identifying financial identifiers.
- A **before-tool-call guardrail** inspects every tool invocation the agent proposes before the tool executes. Read-only calls (`getBalance`, `listTransactions`, `groupByCategory`) pass through immediately. Write calls (`transferFunds`, `payBill`) are validated against a set of rules: the amount must not exceed the account's available balance, the destination must be an account the user controls, and the requesting principal must match the session subject. A failed check blocks the tool call and returns a structured refusal to the agent; the agent incorporates the refusal into its answer without retrying the blocked operation.

The blueprint shows that a single-agent pattern is not a permission to omit governance: two independent checks sit on either side of every sensitive operation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a natural-language query into the **Query** input (e.g., "How much did I spend on groceries last month?" or "Transfer $200 to my savings account").
2. The user selects which **account set** to use from a dropdown (three seeded account sets: individual-checking, joint-household, small-business) or leaves it at the default.
3. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — a small badge shows the count of PII tokens replaced.
5. Within ~10–30 s, the agent returns an `AssistantResponse`. The card transitions through `ANSWERING` to `ANSWERED`. The response appears: a natural-language answer paragraph, a structured data block (e.g., a category-spend table), and a tool-call trace showing which tools fired.
6. If the query triggered a write tool, the card also shows a `WriteOutcome` sub-section: `APPROVED` (tool executed), `BLOCKED` (guardrail rejected), or `ERROR` (tool faulted).
7. The user can submit another query; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → sanitized → answering → answered / failed. Source of truth. | `QueryEndpoint`, `TransactionSanitizer`, `QueryWorkflow` | `QueryView` |
| `TransactionSanitizer` | `Consumer` | Subscribes to `QuerySubmitted` events; tokenises PII in transaction data; calls `QueryEntity.attachSanitized`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitSanitizedStep` → `answerStep` → `recordStep`. | started by `TransactionSanitizer` once sanitized event lands | `FinanceAssistantAgent`, `QueryEntity` |
| `FinanceAssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives account context and the sanitized query; calls finance tools; returns `AssistantResponse`. | invoked by `QueryWorkflow` | returns answer |
| `FinanceTools` | `AgentTool` provider | Five tools: `getBalance`, `listTransactions`, `groupByCategory`, `transferFunds`, `payBill`. Write tools gated by `WriteGuardrail`. | invoked by `FinanceAssistantAgent` | bank simulation state |
| `WriteGuardrail` | guardrail hook | Validates every write-tool invocation (before-tool-call) against balance, ownership, and principal rules. | called on each `transferFunds` / `payBill` intent | pass-through or structured refusal |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record AccountRef(String accountId, String label, String type) {}
// type: CHECKING | SAVINGS | CREDIT | BUSINESS

record TransactionRecord(
    String transactionId,
    String accountId,
    String rawDescription,   // pre-sanitization; audit-only
    String merchant,
    String category,
    BigDecimal amount,
    String currency,
    LocalDate date
) {}

record SanitizedContext(
    List<AccountRef> accounts,
    List<SanitizedTransaction> transactions,
    int piiTokensReplaced
) {}

record SanitizedTransaction(
    String transactionId,
    String accountId,
    String description,      // PII-redacted form
    String merchant,
    String category,
    BigDecimal amount,
    String currency,
    LocalDate date
) {}

record QueryRequest(
    String queryId,
    String queryText,
    String accountSetId,
    List<TransactionRecord> rawTransactions,
    List<AccountRef> accounts,
    String principal,
    Instant submittedAt
) {}

record ToolCallRecord(
    String toolName,
    String arguments,        // JSON
    String result,           // JSON or refusal reason
    WriteOutcome outcome,
    Instant calledAt
) {}
enum WriteOutcome { NOT_A_WRITE, APPROVED, BLOCKED, ERROR }

record AssistantResponse(
    String answer,
    @Nullable String structuredData,    // JSON table or null
    List<ToolCallRecord> toolTrace,
    Instant answeredAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<SanitizedContext> sanitized,
    Optional<AssistantResponse> response,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, SANITIZED, ANSWERING, ANSWERED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `TransactionsSanitized`, `AnsweringStarted`, `AnswerRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ queryText, accountSetId, principal }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Personal Finance Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + age + first 60 chars of query text) and a right pane with the selected query's detail — sanitized context preview (account list + PII-token count), the agent's answer, the structured data block if present, the tool-call trace, and the write-outcome sub-section if a write tool was called.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `TransactionSanitizer` Consumer): tokenises account numbers (replaced with `[ACCT-xxxx]` using the last 4 digits), masks card PAN fragments (`[CARD-xxxx]`), replaces full-name strings with initials (`[NAME-INITIALS]`), and strips sort codes and routing numbers (`[ROUTING-REDACTED]`). Records the count of replacements in `SanitizedContext.piiTokensReplaced`.
- **G1 — before-tool-call guardrail**: runs on every tool invocation the agent proposes. For read-only tools (`getBalance`, `listTransactions`, `groupByCategory`) it passes through immediately. For write tools (`transferFunds`, `payBill`) it enforces: (1) amount ≤ available balance on the source account, (2) destination account is in the user's own `accounts` list, (3) the `principal` on the `QueryRequest` matches the session subject in the tool call arguments. Any failed check returns a structured `blocked-write` error to the agent. The agent incorporates the block into its natural-language answer; the tool does not execute.

## 9. Agent prompts

- `FinanceAssistantAgent` → `prompts/finance-assistant.md`. The single decision-making LLM. System prompt instructs it to answer the user's finance query using the available tools, to state clearly when a write operation was blocked, and to surface tool results in a structured format.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "How much did I spend on groceries and dining last month?" using the individual-checking account set; within 30 s the agent returns a category-spend table with totals and a plain-language summary.
2. **J2** — User submits "Transfer $200 to my savings"; the before-tool-call guardrail approves (amount ≤ balance, destination is own account); the transfer simulates successfully and the confirmation appears in the tool trace.
3. **J3** — User submits "Transfer $50,000 to my savings" against an account with $1,200 balance; the guardrail blocks the tool call; the agent's answer explains the rejection without attempting the transfer; `WriteOutcome.BLOCKED` appears on the card.
4. **J4** — Raw transactions containing account number `4111-xxxx-xxxx-1234` and the principal's full name are submitted; the LLM call log shows only `[ACCT-1234]` and `[NAME-INITIALS]`; `request.rawTransactions` on the entity retains the original for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named personal-finance-agent demonstrating the single-agent × finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-personal-finance-agent. Java package
io.akka.samples.personalfinanceassistant. Akka 3.6.0. HTTP port 9730.

Components to wire (exactly):

- 1 AutonomousAgent FinanceAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/finance-assistant.md>) and
  .capability(TaskAcceptance.of(ANSWER_FINANCE_QUERY).maxIterationsPerTask(5)).
  Tools registered via .tools(FinanceTools.class). Output: AssistantResponse{answer: String,
  structuredData: String (nullable JSON), toolTrace: List<ToolCallRecord>, answeredAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On a write-tool block the agent
  loop receives the structured refusal and incorporates it into the answer on the same iteration;
  no retry of the blocked call.

- 1 AgentTool provider FinanceTools declaring five tools:
  * getBalance(accountId: String) -> BalanceResult{accountId, available, currency}
  * listTransactions(accountId: String, fromDate: LocalDate, toDate: LocalDate) -> List<SanitizedTransaction>
  * groupByCategory(transactions: List<SanitizedTransaction>) -> List<CategoryTotal{category, total, count}>
  * transferFunds(fromAccountId: String, toAccountId: String, amount: BigDecimal, note: String) -> TransferResult{transactionId, status}
  * payBill(accountId: String, payee: String, amount: BigDecimal, reference: String) -> BillResult{confirmationId, status}
  All tools read from / mutate a simulated in-memory account store seeded from
  src/main/resources/sample-events/accounts.json and transactions.jsonl.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitSanitizedStep — polls QueryEntity.getQuery every 1s; on query.sanitized().isPresent()
    advances to answerStep. WorkflowSettings.stepTimeout 15s.
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    FinanceAssistantAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions(formatQuery(query.request.queryText, query.sanitized))
    ) — returns a taskId, then forTask(taskId).result(ANSWER_FINANCE_QUERY) to fetch
    AssistantResponse. On success calls QueryEntity.recordAnswer(response).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * recordStep — calls QueryEntity.finish(). WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, sanitized: Optional<SanitizedContext>,
  response: Optional<AssistantResponse>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED,
  SANITIZED, ANSWERING, ANSWERED, FAILED. Events: QuerySubmitted{request},
  TransactionsSanitized{sanitized}, AnsweringStarted{}, AnswerRecorded{response},
  QueryFailed{reason}. Commands: submit, attachSanitized, markAnswering, recordAnswer,
  finish, fail, getQuery. emptyState() returns Query.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer TransactionSanitizer subscribed to QueryEntity events; on QuerySubmitted runs
  a tokenisation pipeline over rawTransactions: replaces account numbers with [ACCT-last4],
  masks card PAN fragments with [CARD-last4], replaces full-name strings with [NAME-INITIALS],
  strips routing/sort codes with [ROUTING-REDACTED]; counts total replacements; builds
  SanitizedContext; calls QueryEntity.attachSanitized(sanitized). After attachSanitized lands,
  the same Consumer starts a QueryWorkflow with id = "query-" + queryId.

- 1 View QueryView with row type QueryRow (mirrors Query minus request.rawTransactions —
  the audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {queryText, accountSetId, principal};
    mints queryId; calls QueryEntity.submit; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- FinanceTasks.java declaring one Task<R> constant: ANSWER_FINANCE_QUERY =
  Task.name("Answer finance query").description("Use the available finance tools to answer
  the user's question and return an AssistantResponse").resultConformsTo(AssistantResponse.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records AccountRef, TransactionRecord, SanitizedContext, SanitizedTransaction,
  QueryRequest, ToolCallRecord, WriteOutcome, AssistantResponse, Query, QueryStatus.

- WriteGuardrail.java implementing the before-tool-call hook. On write-tool invocations,
  reads the proposed arguments, checks the three rules listed in eval-matrix.yaml G1, and
  either passes the invocation through or returns Guardrail.reject(<structured-block-reason>).
  Read-only tool calls pass through without inspection.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9730 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. FinanceAssistantAgent.definition() binds the configured
  provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent override
  pattern from the akka-context docs.

- src/main/resources/sample-events/accounts.json with 3 seeded account sets:
  individual-checking (2 accounts: checking + savings), joint-household (3 accounts:
  joint-checking, savings, credit), small-business (3 accounts: business-checking,
  payroll, credit-line). Each account carries id, label, type, available balance, currency.

- src/main/resources/sample-events/transactions.jsonl with 60 seeded transactions spread
  across the three account sets (20 per set), covering at least 6 categories (groceries,
  dining, utilities, transport, entertainment, transfers). Each transaction contains 1–2
  PII-like strings (account number references, a full-name in the description) so S1 has
  work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for finance-analysis domain.
  data.data_classes.pii = true, pii_handled_by_sanitizer_before_llm = true,
  decisions.authority_level = act-on-behalf (write tools execute real side-effects in sim),
  oversight.human_in_loop = false (assistant mode — user reviews answer, not a separate
  approver), failure.failure_modes including "blocked-write-not-surfaced", "pii-leakage-to-llm",
  "wrong-account-transfer", "balance-read-stale"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/finance-assistant.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Personal Finance Assistant",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with sanitized context preview,
  answer, structured-data block, tool-call trace, write-outcome sub-section).
  Browser title exactly: <title>Akka Sample: Personal Finance Assistant</title>. No subtitle
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
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json and deserialises pseudo-randomly per
  queryId (seedFor(queryId)).
- Per-task mock-response shapes for THIS blueprint:
    answer-finance-query.json — 8 AssistantResponse entries:
      - 3 read-only answers (spending summaries with structuredData category tables,
        balance lookups, transaction lists). Each has a populated toolTrace showing
        getBalance / listTransactions / groupByCategory calls with realistic arguments
        and results.
      - 3 write-approved answers (transferFunds or payBill calls approved by guardrail,
        confirmation in tool trace, WriteOutcome.APPROVED).
      - 2 write-blocked answers (transferFunds blocked by guardrail — one over-balance,
        one wrong-account — WriteOutcome.BLOCKED on the affected tool call, agent's
        answer text explains the rejection).
- MockModelProvider.seedFor(queryId) makes per-query selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FinanceAssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion FinanceTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (answerStep
  90s, awaitSanitizedStep 15s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>.
- Lesson 7: FinanceTasks.java with ANSWER_FINANCE_QUERY = Task.name(...).description(...)
  .resultConformsTo(AssistantResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9730 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: there is exactly ONE AutonomousAgent (FinanceAssistantAgent).
  No second LLM call anywhere in the stack.
- Write tools pass through WriteGuardrail before execution. Read tools bypass the guardrail.
  This separation MUST be encoded in the tool registration, not as a runtime if-statement
  inside the tool body.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as a manual check inside the tool implementations.
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
