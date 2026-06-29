# SPEC — text-to-sql-guarded

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Advanced Text-to-SQL Workflow.
**One-line pitch:** A user submits a natural-language financial question; one `SqlAgent` walks it through three task phases — **PARSE** the question into a validated SQL statement, **QUERY** the financial database with that statement, **FORMAT** the result set into a structured report — with each phase gated on the prior phase's recorded output, destructive SQL blocked before execution, and PII redacted from every result row.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a finance-analysis domain. One `SqlAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PARSE task's typed output (a `ParsedQuery` carrying the generated SQL) becomes the QUERY task's instruction context; the QUERY task's typed output (a `RawResult` carrying column names and row data) becomes the FORMAT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`PARSE` / `QUERY` / `FORMAT`) and the current `QueryEntity` status. A QUERY-phase tool called while the entity has not yet recorded `QuestionParsed` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. The guardrail is the mechanism that enforces the deeper pipeline property: a phase's tool set is unreachable until its preconditions hold.
- An **automatic safety halt** runs inside `parseStep` immediately after the agent returns a `ParsedQuery`. A deterministic `SqlSafetyInspector` scans the generated SQL for DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, and GRANT keywords. If any are found, the halt records `SafetyHaltFired{sql, pattern}` on the entity and ends the workflow — the database is never contacted.
- A **PII sanitizer** runs inside `formatStep` on the raw result set before it is written onto the entity. A `PiiSanitizer` applies regex-based redaction rules to replace raw SSNs (pattern `\d{3}-\d{2}-\d{4}`), account numbers (9–12 digit sequences that appear in configured column names), and any name that matches the in-process name lookup table. Redacted rows, not raw rows, are stored and surfaced in the API and UI.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce phase isolation, and the governance mechanisms are independent: the guardrail does not check SQL safety; the safety halt does not check phase order; the sanitizer does not check either.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **financial question** into the input (or picks one of three seeded questions — `What were the top 10 vendors by spend last quarter?`, `Show me accounts payable aging over 90 days`, `List all employees with salary above the department average`).
2. The user clicks **Run query**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the agent has been handed the PARSE task.
4. Within ~10–20 s the card reaches `PARSED`. The right pane shows the generated SQL in a code block. If a destructive pattern was detected, the card transitions directly to `HALTED` with the halt reason displayed instead.
5. Within ~10–20 s more the card reaches `QUERIED`. The right pane shows the raw column list and a row count. The raw rows are never displayed — only the post-sanitization report is shown.
6. Within ~10–20 s more the card reaches `FORMATTED`. The right pane shows the full typed `QueryReport` — question, SQL used, redaction summary (count of PII fields masked), a formatted results table with sanitized values, and a one-line narrative.
7. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: created → parsing → parsed → querying → queried → formatting → formatted → halted / failed. Source of truth. | `QueryEndpoint`, `QueryPipelineWorkflow` | `QueryView` |
| `QueryPipelineWorkflow` | `Workflow` | One workflow per queryId. Steps: `parseStep` → `queryStep` → `formatStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `parseStep` also runs `SqlSafetyInspector`; if the halt fires the workflow writes `SafetyHaltFired` and ends without reaching `queryStep`. | started by `QueryEndpoint` after `CREATED` | `SqlAgent`, `QueryEntity` |
| `SqlAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `SqlTasks.java`: `PARSE_QUESTION` → `ParsedQuery`, `EXECUTE_QUERY` → `RawResult`, `FORMAT_RESULTS` → `QueryReport`. Each task is registered with the phase-appropriate function tools. | invoked by `QueryPipelineWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `resolveSchema(question)` and `generateSql(question, schema)`. Reads the in-process schema registry for table and column metadata; produces SQL without executing it. | called from PARSE task | returns `ParsedQuery` |
| `QueryTools` | function-tools class | Implements `executeSql(sql)` and `describeColumns(sql)`. Executes against the in-process H2 database seeded with synthetic financial data. | called from QUERY task | returns `RawResult` |
| `FormatTools` | function-tools class | Implements `narrativeSummary(result)` and `tableLayout(result)`. Pure in-memory transformations. | called from FORMAT task | returns `QueryReport` |
| `SqlGuardrail` | `before-tool-call` guardrail (registered on `SqlAgent`) | Reads the in-flight task's declared phase and the current `QueryEntity.status`. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `SqlSafetyInspector` | plain class (no Akka primitive) | Deterministic halt check. Input: `ParsedQuery.sql`. Output: `SafetyDecision{safe, matchedPattern}`. Patterns: DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, GRANT (case-insensitive, word-boundary). | called from `parseStep` after agent returns | accept or halt |
| `PiiSanitizer` | plain class (no Akka primitive) | Runs inside `formatStep` before `FormatTools`. Input: `RawResult`. Output: `SanitizedResult{rows, redactionCount, redactedFields}`. SSN regex, account-number column names, in-process name table. | called from `formatStep` | returns sanitized rows |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SchemaTable(String tableName, List<String> columnNames, String description) {}

record ParsedQuery(
    String question,
    String sql,
    List<SchemaTable> tablesReferenced,
    Instant parsedAt
) {}

record ResultRow(Map<String, String> cells) {}

record RawResult(
    List<String> columns,
    List<ResultRow> rows,
    int rowCount,
    Instant queriedAt
) {}

record RedactionSummary(int redactionCount, List<String> redactedFields) {}

record SanitizedResult(
    List<String> columns,
    List<ResultRow> rows,
    RedactionSummary redactionSummary
) {}

record QueryReport(
    String question,
    String sqlUsed,
    String narrative,
    SanitizedResult result,
    Instant formattedAt
) {}

record SafetyDecision(boolean safe, Optional<String> matchedPattern) {}

record GuardrailRejection(String phase, String tool, String reason, Instant rejectedAt) {}

record QueryRecord(
    String queryId,
    Optional<String> question,
    Optional<ParsedQuery> parsedQuery,
    Optional<RawResult> rawResult,
    Optional<QueryReport> report,
    Optional<String> haltReason,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum QueryStatus {
    CREATED, PARSING, PARSED, QUERYING, QUERIED,
    FORMATTING, FORMATTED, HALTED, FAILED
}
```

Events on `QueryEntity`: `QueryCreated`, `ParseStarted`, `QuestionParsed`, `SafetyHaltFired`, `QueryStarted`, `QueryExecuted`, `FormatStarted`, `ResultsFormatted`, `GuardrailRejected`, `QueryFailed`.

Every nullable lifecycle field on the `QueryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Advanced Text-to-SQL Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of queries (status pill + question preview + age) and a right pane with the selected query's detail — question, generated SQL code block, raw column list, redaction summary, formatted results table, narrative, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (phase-gate)**: `SqlGuardrail` is registered on `SqlAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `SqlPhase.PARSE`, `SqlPhase.QUERY`, `SqlPhase.FORMAT`) and the current `QueryEntity.status` for the query the task is bound to. The accept rule is precise: `PARSE` tools require `status ∈ {CREATED, PARSING}`; `QUERY` tools require `status ∈ {PARSED, QUERYING}` AND `parsedQuery.isPresent()` AND `parsedQuery.get().sql` passed the safety inspector; `FORMAT` tools require `status ∈ {QUERIED, FORMATTING}` AND `rawResult.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event. The agent loop retries within its 4-iteration budget.
- **H1 — automatic safety halt**: runs inside `parseStep` immediately after the agent returns a `ParsedQuery`. `SqlSafetyInspector` scans the SQL for the pattern set `{DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, GRANT}` (case-insensitive, full-word match). On a match, the workflow writes `SafetyHaltFired{sql, matchedPattern}` on the entity, transitions status to `HALTED`, and ends — the `queryStep` is never reached. The halt is blocking and not retryable within the same workflow run; the user must submit a new question.
- **S1 — PII sanitizer**: runs inside `formatStep` on the `RawResult` before `FormatTools` sees it. `PiiSanitizer` applies three redaction rules: (1) SSN columns — any cell matching `\d{3}-\d{2}-\d{4}` is replaced with `***-**-****`; (2) account-number columns — cells in columns named `account_number`, `acct_no`, `account_id` (exact match) are replaced with `[REDACTED]` if the value is a 9–12 digit sequence; (3) name columns — cells in columns named `employee_name`, `customer_name`, `full_name` are checked against an in-process name lookup table built from the seeded H2 data; matches are replaced with `[NAME REDACTED]`. The `redactionSummary` field records count and field names for audit. The sanitized result, not the raw result, is written onto the entity.

## 9. Agent prompts

- `SqlAgent` → `prompts/sql-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, never invent table names or column names not present in the resolved schema, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded question `What were the top 10 vendors by spend last quarter?`; within 60 s the query reaches `FORMATTED` with a non-empty SQL statement, ≥ 1 table referenced, a `QueryReport` with ≥ 1 result row, and a non-zero redaction count if any PII-tagged columns appear.
2. **J2** — The mock LLM generates SQL containing `DELETE FROM transactions`. `SqlSafetyInspector` fires; `SafetyHaltFired` lands on the entity with `matchedPattern = "DELETE"`; the card transitions to `HALTED`; the database is never contacted. The UI shows the halt reason on the card.
3. **J3** — A query returns rows where the `employee_name` column contains a real name from the seeded lookup table. The `PiiSanitizer` replaces the name with `[NAME REDACTED]`; the `QueryReport` carries the redacted value and `redactionCount >= 1`; the raw name never appears in the API response.
4. **J4** — A phase-order violation (QUERY-phase tool `executeSql` called during the PARSE phase on the mock-LLM path) is caught by `SqlGuardrail`, logged as `GuardrailRejected{phase: "PARSE", tool: "executeSql", reason: "..."}`, and the agent retries. The briefing completes correctly. The UI rejection-log strip shows the one rejected call.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named text-to-sql-guarded demonstrating the sequential-pipeline x
finance-analysis cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact sequential-pipeline-finance-analysis-text-to-sql-guarded.
Java package io.akka.samples.advancedtexttosqlworkflow. Akka 3.6.0. HTTP port 9790.

Components to wire (exactly):

- 1 AutonomousAgent SqlAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/sql-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  PARSE, QUERY, and FORMAT tool sets are ALL registered on the agent; phase gating is the job
  of SqlGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call guardrail
  (SqlGuardrail) is registered on the agent via the agent's guardrail-configuration block. On
  guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow QueryPipelineWorkflow per queryId with three primary steps plus error and halt:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(SqlAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions("Question: " + question + "\nPhase: PARSE\nUse resolveSchema then
      generateSql to produce a read-only SQL statement for this question.")
        .metadata("queryId", queryId)
        .metadata("phase", "PARSE")
        .taskType(SqlTasks.PARSE_QUESTION)
    ). Reads forTask(taskId).result(PARSE_QUESTION) to get ParsedQuery. BEFORE recording
    QuestionParsed, calls SqlSafetyInspector.inspect(parsedQuery.sql). If the decision is
    not safe: writes SafetyHaltFired{sql, matchedPattern} on the entity and transitions via
    haltStep (writes status HALTED, ends). If safe: writes QueryEntity.recordParsed(parsedQuery).
    WorkflowSettings.stepTimeout 60s.
  * queryStep — emits QueryStarted, then runSingleTask with TaskDef.instructions
    (formatQueryContext(parsedQuery, question)) and metadata.phase = "QUERY", taskType
    EXECUTE_QUERY. Writes QueryEntity.recordQueryResult(rawResult). stepTimeout 60s.
  * formatStep — emits FormatStarted, then calls PiiSanitizer.sanitize(rawResult) to get
    SanitizedResult, then runSingleTask with TaskDef.instructions(
    formatFormatContext(sanitizedResult, parsedQuery, question)) and metadata.phase = "FORMAT",
    taskType FORMAT_RESULTS. Writes QueryEntity.recordReport(report). stepTimeout 60s.
  * haltStep — writes SafetyHaltFired (already recorded in parseStep), ends. stepTimeout 5s.
  * error step — writes QueryFailed, ends. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(QueryPipelineWorkflow::error). The error step writes
  QueryFailed and ends.

- 1 EventSourcedEntity QueryEntity (one per queryId). State QueryRecord{queryId,
  question: Optional<String>, parsedQuery: Optional<ParsedQuery>,
  rawResult: Optional<RawResult>, report: Optional<QueryReport>,
  haltReason: Optional<String>, status: QueryStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  QueryStatus enum: CREATED, PARSING, PARSED, QUERYING, QUERIED, FORMATTING, FORMATTED,
  HALTED, FAILED. Events: QueryCreated{question}, ParseStarted, QuestionParsed{parsedQuery},
  SafetyHaltFired{sql, matchedPattern}, QueryStarted, QueryExecuted{rawResult},
  FormatStarted, ResultsFormatted{report}, GuardrailRejected{phase, tool, reason},
  QueryFailed{reason}. Commands: create, startParse, recordParsed, recordHalt, startQuery,
  recordQueryResult, startFormat, recordReport, recordGuardrailRejection, fail, getQuery.
  emptyState() returns QueryRecord.initial("") with all Optional fields as Optional.empty()
  and no commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View QueryView with row type QueryRow that mirrors QueryRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes QueryEntity events. ONE query
  getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question}; mints queryId; calls
    QueryEntity.create(question); then starts QueryPipelineWorkflow with id
    "pipeline-" + queryId; returns {queryId}), GET /queries (list from getAllQueries, sorted
    newest-first), GET /queries/{id} (one row), GET /queries/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- SqlTasks.java declaring three Task<R> constants:
    PARSE_QUESTION = Task.name("Parse question").description("Resolve the financial schema
      and generate a read-only SQL statement for the user question").resultConformsTo(
      ParsedQuery.class);
    EXECUTE_QUERY = Task.name("Execute query").description("Execute the validated SQL against
      the financial database and return the raw result set").resultConformsTo(
      RawResult.class);
    FORMAT_RESULTS = Task.name("Format results").description("Produce a narrative summary and
      formatted table from the sanitized result set").resultConformsTo(QueryReport.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- SqlPhase.java — enum {PARSE, QUERY, FORMAT}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "resolveSchema", phase = SqlPhase.PARSE)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- ParseTools.java — @FunctionTool resolveSchema(String question) -> List<SchemaTable> reading
  from the in-process schema registry (one SchemaTable per H2 table, loaded at startup from
  src/main/resources/sample-data/schema/*.json); @FunctionTool generateSql(String question,
  List<SchemaTable> schema) -> ParsedQuery constructing the SQL string and tablesReferenced
  list.

- QueryTools.java — @FunctionTool executeSql(String sql) -> RawResult running the SQL against
  the in-process H2 database and returning columns + rows; @FunctionTool
  describeColumns(String sql) -> List<String> returning the projected column list without
  executing.

- FormatTools.java — @FunctionTool narrativeSummary(SanitizedResult result) -> String
  producing a 1–3 sentence plain-English summary; @FunctionTool tableLayout(
  SanitizedResult result) -> QueryReport assembling the final report with the narrative and
  sanitized rows.

- SqlGuardrail.java — implements the before-tool-call hook. Reads the candidate tool call's
  @FunctionTool.phase attribute, looks up the QueryEntity status by queryId (carried in the
  TaskDef metadata), applies the accept matrix from Section 8, and either passes or returns
  Guardrail.reject("phase-violation: <tool> requires <precondition>, saw <status>"). On
  reject ALSO calls QueryEntity.recordGuardrailRejection(phase, tool, reason).

- SqlSafetyInspector.java — pure deterministic logic (no LLM). Input: String sql. Output:
  SafetyDecision{safe, matchedPattern: Optional<String>}. Scans for the word-boundary
  pattern set {DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, GRANT} case-insensitively.
  Returns safe=false and the first matched pattern on any hit; safe=true otherwise. No network
  call, no entity access — purely synchronous in-process check.

- PiiSanitizer.java — pure deterministic logic (no LLM). Input: RawResult. Output:
  SanitizedResult. Three redaction rules: (1) SSN regex on all cells, (2) account-number
  column name match + digit-sequence check, (3) name column name match + in-process name
  lookup built from seeded H2 data. Tracks redactionCount and redactedFields for the
  summary.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9790 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/questions.jsonl with 5 seeded question lines including
  the three surfaces named in J1–J4 plus two extras.

- src/main/resources/sample-data/schema/*.json — three files carrying SchemaTable entries
  for the seeded H2 schema (transactions, vendors, employees, accounts_payable tables).

- src/main/resources/sample-data/seed.sql — DDL and INSERT statements for the H2 database.
  Read-only columns: all. PII-tagged columns: employees.employee_name, employees.ssn,
  transactions.account_number.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true (result rows may
  contain employee SSNs and account numbers), decisions.authority_level = recommend-only
  (the report is advisory), oversight.human_in_loop = true, operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "destructive-sql-execution", "pii-exposure", "phase-violation", "hallucinated-table",
  "schema-mismatch"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/sql-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Advanced Text-to-SQL Workflow",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with question header, SQL code
  block, redaction summary, results table, narrative, rejection-log strip). Browser title
  exactly: <title>Akka Sample: Advanced Text-to-SQL Workflow</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; gone when the
        session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per call
  (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes:
    parse-question.json — 5 ParsedQuery entries, each with a SELECT SQL statement and
      tablesReferenced list. Plus 1 deliberately DESTRUCTIVE entry whose sql starts with
      "DELETE FROM transactions" — SqlSafetyInspector fires; J2 is reproducible on the 2nd
      submission of every 4th queryId modulo seed.
    execute-query.json — 5 RawResult entries paired one-to-one with the parse entries, each
      carrying 3–8 ResultRow maps. The employees-related entry deliberately includes one row
      with employee_name = "Jane Smith" and ssn = "123-45-6789" — J3 verifies the sanitizer
      fires.
    format-results.json — 5 QueryReport entries with narratives referencing the paired
      sanitized result.
  Plus 1 deliberately PHASE-VIOLATING parse entry whose tool_calls array starts with
  executeSql (a QUERY-phase tool called during PARSE) — the guardrail rejects it; J4 verifies
  this.
- MockModelProvider.seedFor(queryId) makes per-query selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SqlAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SqlTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, queryStep 60s, formatStep 60s, haltStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on QueryRecord is Optional<T>.
- Lesson 7: SqlTasks.java with PARSE_QUESTION, EXECUTE_QUERY, FORMAT_RESULTS constants is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9790 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in prose.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (SqlAgent). SqlSafetyInspector
  and PiiSanitizer are plain Java classes — no LLM call.
- The sequential-pipeline invariant: all three phase tool sets are registered on the agent;
  SqlGuardrail is the runtime gate — do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: parseStep writes ParsedQuery onto the
  entity, queryStep reads it and builds the QUERY task's instruction context from it,
  formatStep reads both. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
