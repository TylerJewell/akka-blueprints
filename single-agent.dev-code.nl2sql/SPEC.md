# SPEC — nl2sql

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** NL2SQL.
**One-line pitch:** A user asks a natural-language question; one AI agent translates it into a read-only SQL query, executes it against a PostgreSQL database, and returns a structured result — column names, typed rows, and a plain-English summary.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `SqlQueryAgent` (AutonomousAgent) carries the entire translation and execution decision; the surrounding components only prepare its schema context and audit its output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs on every tool invocation inside the agent's loop — before the database tool fires. It inspects the generated SQL for write keywords (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, `CREATE`, `MERGE`) and for broad `SELECT *` without a `LIMIT`. A query matching any forbidden pattern is rejected, and the agent must narrow or correct its query within its iteration budget.
- An **automatic-safety-halt** is a harder stop: if the generated SQL contains a write or DDL keyword, the task is terminated immediately regardless of iteration count. This is distinct from the guardrail — the guardrail gives the agent a chance to self-correct; the halt does not.

The blueprint shows that a database-access agent requires layered safety: a soft guardrail for self-correctable problems, and a hard halt for unambiguously dangerous statements.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** textarea (or picks one of three seeded examples — monthly revenue by product, top customers by order count, average fulfillment time by region).
2. The user picks a **Schema context** from a dropdown (orders, customers, fulfillment) or selects `full schema` to give the agent all tables.
3. The user clicks **Run query**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SCHEMA_ATTACHED` — the relevant schema fragment is visible in the card detail.
5. Within ~5–30 s, the workflow's `translateAndRunStep` completes. The card transitions to `TRANSLATING` then `RESULT_READY`. The result appears: the generated SQL in a monospace block, a row count badge, and a paginated data table (column headers + typed cell values).
6. Within ~1 s of the result, the `scoreStep` finishes. The card shows a **quality score** chip (1–5) plus a one-line rationale describing whether the result is well-formed and row-count reasonable.
7. The user can submit another question; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → schema_attached → translating → result_ready → scored. Source of truth. | `QueryEndpoint`, `SchemaRegistryConsumer`, `QueryWorkflow` | `QueryView` |
| `SchemaRegistryConsumer` | `Consumer` | Subscribes to `QuerySubmitted` events; fetches relevant schema fragment from the in-process registry; calls `QueryEntity.attachSchema`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitSchemaStep` → `translateAndRunStep` → `scoreStep`. | started by `SchemaRegistryConsumer` once schema lands | `SqlQueryAgent`, `QueryEntity` |
| `SqlQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the schema fragment and the user question; uses `SchemaLookupTool` and `QueryExecutionTool`; returns `QueryResult`. | invoked by `QueryWorkflow` | returns result |
| `SqlSafetyGuardrail` | supporting class | before-tool-call hook on `SqlQueryAgent`; rejects unsafe SQL before the tool fires. | bound to `SqlQueryAgent` | — |
| `QueryResultScorer` | supporting class | Deterministic rule-based scorer; runs in `scoreStep`; no LLM call. | invoked by `QueryWorkflow` | returns `ScoreResult` |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SchemaColumn(String columnName, String dataType, boolean nullable) {}

record SchemaTable(
    String tableName,
    String description,
    List<SchemaColumn> columns
) {}

record SchemaFragment(
    List<SchemaTable> tables,
    String contextName
) {}

record QueryRequest(
    String queryId,
    String question,
    String schemaContext,
    String submittedBy,
    Instant submittedAt
) {}

record QueryResult(
    String generatedSql,
    List<String> columnNames,
    List<List<Object>> rows,
    int rowCount,
    String summary,
    Instant executedAt
) {}

record ScoreResult(
    int score,             // 1..5
    String rationale,
    Instant scoredAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<SchemaFragment> schema,
    Optional<QueryResult> result,
    Optional<ScoreResult> score,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, SCHEMA_ATTACHED, TRANSLATING, RESULT_READY, SCORED, HALTED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `SchemaAttached`, `TranslationStarted`, `ResultRecorded`, `QueryScored`, `QueryHalted`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question, schemaContext, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: NL2SQL</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + score chip + age) and a right pane with the selected query's detail — the user question, schema fragment preview, generated SQL block, result data table, and score chip with rationale.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`): runs on every tool call inside `SqlQueryAgent`. Inspects the `sql` argument of `QueryExecutionTool` calls. Rejects any SQL containing write or DDL keywords (`INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, `CREATE`, `MERGE`) and any `SELECT *` without an explicit `LIMIT`. On rejection, returns a structured error naming the forbidden pattern; the agent loop consumes one iteration and retries with a corrected query. The guardrail does not terminate the task; it gives the agent a chance to self-correct.
- **H1 — automatic-safety-halt** (`halt`, `automatic-safety-halt`): a harder gate that terminates the task immediately on any write or DDL keyword in the generated SQL. Unlike G1, this does not allow retry — the task ends, the entity transitions to `HALTED`, and the user sees a clear explanation. The halt fires after the guardrail on the same tool-call event; if the guardrail misses a write pattern (e.g. the agent produced a syntactically obscured form), the halt catches it.

## 9. Agent prompts

- `SqlQueryAgent` → `prompts/sql-query-agent.md`. The single decision-making LLM. System prompt instructs it to use the schema lookup tool to understand the relevant tables and columns, generate a SELECT query, execute it, and return one `QueryResult` with the SQL, rows, column names, row count, and a plain-English summary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks "what is total revenue per product this month?"; within 30 s the result appears with a generated SQL SELECT and a populated data table.
2. **J2** — The agent generates a `DELETE` statement on the first iteration (mock LLM path) — the safety halt fires immediately; the task terminates; the entity transitions to `HALTED`; the database is never touched.
3. **J3** — The agent returns a result with zero rows — the scorer assigns score 2 with the rationale "Query returned no rows; confirm schema context is correct."
4. **J4** — The agent generates `SELECT *` without a LIMIT on the first iteration — the guardrail rejects it; the agent adds explicit column names and a `LIMIT 1000`; the second iteration succeeds.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named nl2sql demonstrating the single-agent × dev-code cell. Requires
Docker Desktop or Docker Engine with Compose v2 (for the PostgreSQL container). Maven
group io.akka.samples. Maven artifact single-agent-dev-code-nl2sql. Java package
io.akka.samples.naturallanguagequeriesonpostgresql. Akka 3.6.0. HTTP port 9643.

Components to wire (exactly):

- 1 AutonomousAgent SqlQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/sql-query-agent.md>) and
  .capability(TaskAcceptance.of(RUN_QUERY).maxIterationsPerTask(4)). The task receives
  the user question and schema context as its instruction text. The agent has two tools:
  SchemaLookupTool (looks up a table definition by name from the in-process schema
  registry) and QueryExecutionTool (executes a SQL SELECT against the local Postgres
  container; reads JDBC_URL / JDBC_USER / JDBC_PASSWORD from env; enforces a
  read-only role at the connection level). The agent is configured with a
  before-tool-call guardrail (G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block AND with an automatic-safety-halt (H1 in
  eval-matrix.yaml). Output: QueryResult{generatedSql: String, columnNames:
  List<String>, rows: List<List<Object>>, rowCount: int, summary: String,
  executedAt: Instant}. On halt, the agent's task terminates without returning a
  QueryResult; the workflow's translateAndRunStep catches the HaltException and calls
  QueryEntity.halt(reason).

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitSchemaStep — polls QueryEntity.getQuery every 1 s; on query.schema().isPresent()
    advances to translateAndRunStep. WorkflowSettings.stepTimeout 10 s (schema consumer
    is in-process).
  * translateAndRunStep — emits TranslationStarted, then calls
    componentClient.forAutonomousAgent(SqlQueryAgent.class, "agent-" + queryId)
    .runSingleTask(TaskDef.instructions(formatQuestion(query.request))) to get a
    taskId, then forTask(taskId).result(RUN_QUERY) to fetch the result. On success
    calls QueryEntity.recordResult(result). On HaltException calls
    QueryEntity.halt(reason). WorkflowSettings.stepTimeout 90 s with
    defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).
  * scoreStep — runs a deterministic rule-based QueryResultScorer (NOT an LLM call)
    over the recorded result: checks row count (0 = score 1, 1–1000 reasonable = full
    marks, >10000 = docked), column-name coverage against the schema fragment (expected
    columns present?), SQL complexity (bare SELECT * loses a point). Emits
    QueryScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5 s.
    error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, schema: Optional<SchemaFragment>,
  result: Optional<QueryResult>, score: Optional<ScoreResult>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED,
  SCHEMA_ATTACHED, TRANSLATING, RESULT_READY, SCORED, HALTED, FAILED. Events:
  QuerySubmitted{request}, SchemaAttached{schema}, TranslationStarted{},
  ResultRecorded{result}, QueryScored{score}, QueryHalted{reason}, QueryFailed{reason}.
  Commands: submit, attachSchema, markTranslating, recordResult, recordScore, halt,
  fail, getQuery. emptyState() returns Query.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state
  and Optional.of(...) inside the event-applier.

- 1 Consumer SchemaRegistryConsumer subscribed to QueryEntity events; on QuerySubmitted
  looks up the schemaContext name in an in-process SchemaRegistry (seeded from
  src/main/resources/sample-events/schema-registry.jsonl); builds a SchemaFragment
  containing the relevant tables; calls QueryEntity.attachSchema(schema). After
  attachSchema lands, starts a QueryWorkflow with id = "query-" + queryId.

- 1 View QueryView with row type QueryRow (mirrors Query minus any raw internal state).
  Table updater consumes QueryEntity events. ONE query getAllQueries:
  SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question, schemaContext,
    submittedBy}; mints queryId; calls QueryEntity.submit; returns {queryId}),
    GET /queries (list from getAllQueries, sorted newest-first), GET /queries/{id}
    (one row), GET /queries/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: RUN_QUERY = Task.name("Run query")
  .description("Translate the natural-language question to SQL, execute it against
  the Postgres database, and return a QueryResult").resultConformsTo(QueryResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records SchemaColumn, SchemaTable, SchemaFragment, QueryRequest, QueryResult,
  ScoreResult, Query, QueryStatus.

- SqlSafetyGuardrail.java implementing the before-tool-call hook. Inspects the sql
  argument of QueryExecutionTool calls. Returns Guardrail.reject(<structured-error>)
  on: any write/DDL keyword present (case-insensitive), SELECT * without LIMIT. On
  accept, passes the tool call through. Registered on SqlQueryAgent via the agent's
  guardrail-configuration block.

- SqlSafetyHalt.java implementing the automatic-safety-halt hook. Checks the same
  write/DDL keyword list as SqlSafetyGuardrail. On detection, throws HaltException
  with a clear message naming the detected keyword. Registered on SqlQueryAgent as a
  halt handler. The halt fires after the guardrail on the same tool-call event.

- QueryResultScorer.java — pure deterministic logic (no LLM). Inputs: QueryResult and
  SchemaFragment. Outputs: ScoreResult. Scoring rubric documented in Javadoc.

- SchemaRegistry.java — in-process singleton seeded from schema-registry.jsonl;
  lookup by contextName returns a SchemaFragment. Used by SchemaRegistryConsumer and
  by SqlQueryAgent's SchemaLookupTool.

- docker-compose.yml at the project root. Service: postgres:15-alpine, port 5432,
  env POSTGRES_DB=nl2sql_db POSTGRES_USER=nl2sql_user POSTGRES_PASSWORD=nl2sql_pass,
  volume for init SQL scripts at docker-entrypoint-initdb.d/.

- src/main/resources/db/init.sql — DDL for the seeded schema: orders, order_items,
  customers, products, fulfillment tables; 50 synthetic rows per table via
  INSERT statements.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9643,
  the JDBC connection block reading ${?JDBC_URL} ${?JDBC_USER} ${?JDBC_PASSWORD}
  (defaulting to the docker-compose values for dev mode), and the three model-provider
  blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash)
  reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/schema-registry.jsonl with 3 seeded schema
  contexts: orders (3 tables: orders, order_items, products), customers (2 tables:
  customers, orders), fulfillment (3 tables: fulfillment, orders, products).

- src/main/resources/sample-events/seed-questions.jsonl with 6 natural-language
  questions paired to each schema context: 2 per context, varying complexity (simple
  aggregate, multi-table join).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching Section 8.
  Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (database
  content is business data, not PII by default; deployer overrides as needed),
  decisions.authority_level = automated-execution (the agent executes the query
  without human approval — reads only), oversight.human_in_loop = false (reads are
  low-risk when write prevention is enforced), failure.failure_modes including
  "wrong-table-join", "over-broad-scan", "schema-hallucination", "empty-result-set",
  "unsafe-mutation-attempt"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/sql-query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: NL2SQL", prerequisites
  (including Docker), generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = live list of query cards; right = selected-query detail with question,
  schema fragment preview, generated SQL monospace block, result data table, and score
  chip). Browser title exactly: <title>Akka Sample: NL2SQL</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- Bootstrap.java fails fast with a clear message if JDBC_URL / JDBC_USER /
  JDBC_PASSWORD do not resolve; message must not echo credentials.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes:
    run-query.json — 8 QueryResult entries covering the three schema contexts.
      Each entry has a generatedSql (a valid SELECT for the context), columnNames,
      rows (3–10 rows of plausible synthetic data), rowCount, and a summary sentence.
      Plus 2 deliberately UNSAFE entries (one with DELETE, one with DROP TABLE) — the
      halt fires on both, exercising the H1 path. The mock selects an unsafe entry on
      the FIRST iteration of every 4th query (modulo seed) so J2 is reproducible.
      Plus 1 SELECT * entry without LIMIT — the guardrail rejects it, exercising
      the G1 path (first iteration of every 3rd query modulo seed).
      Plus 1 zero-rows entry (empty rows array) — exercises the J3 score path.
- MockModelProvider.seedFor(queryId) makes per-query selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: SqlQueryAgent extends AutonomousAgent; never downgraded to Agent. Companion
  QueryTasks.java MUST exist.
- Lesson 4: awaitSchemaStep 10 s, translateAndRunStep 90 s, scoreStep 5 s, error 5 s.
- Lesson 6: every Optional<T> lifecycle field on Query uses Optional.empty() in
  emptyState() and Optional.of(...) in event-appliers.
- Lesson 7: QueryTasks.java with RUN_QUERY = Task.name(...).description(...)
  .resultConformsTo(QueryResult.class) is mandatory.
- Lesson 8: model names — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated model ids.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9643 declared in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata in user-facing strings.
- Lesson 12: UI fits 1080 px content column with no horizontal scroll.
- Lesson 13: integration label "Requires Docker" — never T1/T2/T3/T4 in user-visible
  strings.
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND
  themeVariables block.
- Lesson 25: never write key value to disk; record only the reference.
- Lesson 26: tab switching by data-tab / data-panel attribute; exactly five
  <section class="tab-panel"> elements.
- Single-agent invariant: exactly ONE AutonomousAgent (SqlQueryAgent). The scorer is
  rule-based and does NOT make an LLM call.
- Both G1 and H1 are registered on SqlQueryAgent — G1 as a guardrail allowing retry,
  H1 as a halt that terminates the task immediately. Both check the same keyword set
  independently; H1 is the backstop.
- QueryExecutionTool must use a read-only Postgres role (GRANT SELECT; no WRITE).
  The role is provisioned by docker-entrypoint-initdb.d/init.sql.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools. Before running, ensure the Docker Compose Postgres container is up (`docker compose up -d`).

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, Docker not running, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
