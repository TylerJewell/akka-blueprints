# SPEC — text-to-sql-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** TextToSqlAgent.
**One-line pitch:** A user asks a natural-language question about receipt spending; one AI agent translates it into SQL, executes the query against an embedded SQLite receipts table, and returns a structured answer — the generated SQL, the result rows, and a one-sentence prose summary.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `SqlGeneratorAgent` (AutonomousAgent) carries the entire question-answering decision; the surrounding components only prepare its input, guard its tool calls, and audit its output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs inside the agent loop every time the agent is about to invoke the `execute_sql` tool. It inspects the generated SQL for destructive statements (`DROP`, `DELETE`, `UPDATE`, `INSERT`, `TRUNCATE`) and for injection signatures before the database call fires. A rejected SQL causes the agent loop to retry.
- A **PII sanitizer** runs inside a Consumer between the raw query execution and the response returned to the caller — so customer names and email addresses in the result set are redacted before the user ever sees them.

The blueprint shows that even a pure read-side financial query agent warrants defence-in-depth: SQL injection is a real attack vector, and receipt data routinely contains customer identifiers that should not be echoed verbatim into chat interfaces.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a natural-language question into the **Question** input (for example: "What is the total spend per merchant for the last 30 days?" or "Which customer had the highest single receipt amount in Q2?").
2. The user optionally picks one of three seeded question templates from a dropdown.
3. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The question card appears in the live list in `SUBMITTED` state. Within ~10–30 s it transitions to `EXECUTING` — the agent has generated SQL visible in the card detail.
5. If the guardrail fires, the card briefly shows `GUARDRAIL_RETRY` and the rejected SQL is visible with a red badge. The agent retries.
6. Once the database has returned results, the card transitions to `EXECUTED` then `SANITIZED`. The result rows appear in a table; any `[REDACTED-EMAIL]` or `[REDACTED-NAME]` tokens are shown with a muted badge.
7. The user can submit another question; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → generating → executing → executed → sanitized. Source of truth. | `QueryEndpoint`, `ResultSanitizer`, `QueryWorkflow` | `QueryView` |
| `ResultSanitizer` | `Consumer` | Subscribes to `QueryExecuted` events; redacts PII from result rows; calls `QueryEntity.attachSanitized`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `generateStep` → `executeStep` → `awaitSanitizedStep`. | started by `QueryEndpoint` after submit | `SqlGeneratorAgent`, `QueryEntity` |
| `SqlGeneratorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the natural-language question in the task definition and the receipts schema as a task attachment; uses `execute_sql` tool; returns `QueryResult`. | invoked by `QueryWorkflow` | returns result |
| `SqlGuardrail` | before-tool-call hook | Validates generated SQL before `execute_sql` fires. Rejects destructive statements and injection signatures. | `SqlGeneratorAgent` tool calls | agent loop |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record QueryRequest(
    String queryId,
    String question,
    String submittedBy,
    Instant submittedAt
) {}

record GeneratedSql(
    String sql,
    String explanation    // one sentence: why this SQL answers the question
) {}

record ResultRow(Map<String, Object> columns) {}

record RawQueryResult(
    GeneratedSql generatedSql,
    List<ResultRow> rows,
    String summary,       // one-sentence prose answer
    Instant executedAt
) {}

record SanitizedResult(
    List<ResultRow> rows,           // customer names and emails replaced with redaction tokens
    List<String> piiCategoriesFound // e.g. ["customer-name", "email"]
) {}

record QueryResult(
    String queryId,
    Optional<QueryRequest> request,
    Optional<GeneratedSql> generatedSql,
    Optional<RawQueryResult> rawResult,
    Optional<SanitizedResult> sanitized,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, GENERATING, EXECUTING, EXECUTED, SANITIZED, FAILED
}
```

Events on `QueryEntity`: `QuestionSubmitted`, `SqlGenerated`, `ExecutionStarted`, `QueryExecuted`, `ResultSanitized`, `QueryFailed`.

Every nullable lifecycle field on the `QueryResult` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Text-to-SQL Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted questions (status pill + age) and a right pane with the selected query's detail — original question, generated SQL (with a syntax-highlighted code block), guardrail status, sanitized result table, and prose summary.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `execute_sql` tool call inside `SqlGeneratorAgent`. Inspects the generated SQL: rejects any statement that contains a DDL/DML keyword (`DROP`, `DELETE`, `UPDATE`, `INSERT`, `TRUNCATE`, `ALTER`, `CREATE`), any statement containing classic injection signatures (`--`, `'; `, `UNION SELECT`, `OR 1=1`), and any statement referencing tables not in the receipts schema. On rejection, returns a structured error naming the failed check; the agent loop retries within its iteration budget.
- **S1 — PII sanitizer** (`pii`, applied inside `ResultSanitizer` Consumer): after `QueryExecuted`, scans every result row's column values for customer names (matched against the known `customer_name` column) and email addresses (regex), replaces them with `[REDACTED-NAME]` and `[REDACTED-EMAIL]` tokens respectively, and records which categories were found. The raw result rows are preserved on the entity for audit; the sanitized form is what the UI displays.

## 9. Agent prompts

- `SqlGeneratorAgent` → `prompts/sql-generator.md`. The single decision-making LLM. System prompt instructs it to read the attached schema, translate the question into a safe `SELECT` statement using only the receipts schema columns, call the `execute_sql` tool with the generated SQL, and return a `QueryResult` with the rows and a one-sentence summary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks "What is the total amount spent per merchant?"; within 30 s the agent generates valid SQL, the database returns rows, PII is redacted, and the sanitized table appears in the UI.
2. **J2** — The agent on first try generates a `DELETE FROM receipts` statement (mock LLM path); the before-tool-call guardrail rejects it; the agent retries with a `SELECT`; the corrected query executes; the UI never displays the destructive SQL as executed.
3. **J3** — A result set containing `customer_name` and `email` columns has both replaced with redaction tokens before reaching the UI; the raw result on the entity retains the originals for audit.
4. **J4** — A question outside the receipts schema ("What is the weather today?") returns an empty result set with a summary explaining that the question cannot be answered from the available data.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named text-to-sql-agent demonstrating the single-agent × finance-analysis cell.
Runs out of the box (no external services; SQLite is embedded). Maven group io.akka.samples.
Maven artifact single-agent-finance-analysis-text-to-sql-agent.
Java package io.akka.samples.texttosqlagent. Akka 3.6.0. HTTP port 9505.

Components to wire (exactly):

- 1 AutonomousAgent SqlGeneratorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/sql-generator.md>) and
  .capability(TaskAcceptance.of(GENERATE_AND_RUN_QUERY).maxIterationsPerTask(4)).
  The agent has one registered tool: execute_sql(sql: String) → List<ResultRow>, which
  executes the given SQL against the embedded SQLite receipts.db. The tool is defined as a
  method on the agent annotated @Tool. The task receives the natural-language question as its
  instruction text and the receipts schema DDL as a task ATTACHMENT named "schema.sql"
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: QueryResult{queryId, generatedSql, rawResult, sanitized, status}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries the tool call within its 4-iteration budget.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * generateStep — emits SqlGenerated if the agent successfully produces a query. Calls
    componentClient.forAutonomousAgent(SqlGeneratorAgent.class, "sql-" + queryId)
    .runSingleTask(
      TaskDef.instructions(question)
        .attachment("schema.sql", loadSchemaDdl())
    ) — returns a taskId, then forTask(taskId).result(GENERATE_AND_RUN_QUERY).
    On success calls QueryEntity.recordResult(rawResult). WorkflowSettings.stepTimeout 90s
    with defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).
  * awaitSanitizedStep — polls QueryEntity.getQuery every 1s up to 15s; on
    query.sanitized().isPresent() transitions to done. WorkflowSettings.stepTimeout 15s.
  * error step — calls QueryEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State QueryResult{queryId: String,
  request: Optional<QueryRequest>, generatedSql: Optional<GeneratedSql>,
  rawResult: Optional<RawQueryResult>, sanitized: Optional<SanitizedResult>,
  status: QueryStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  QueryStatus enum: SUBMITTED, GENERATING, EXECUTING, EXECUTED, SANITIZED, FAILED.
  Events: QuestionSubmitted{request}, SqlGenerated{sql, explanation}, ExecutionStarted{},
  QueryExecuted{rawResult}, ResultSanitized{sanitized}, QueryFailed{reason}.
  Commands: submit, recordSql, markExecuting, recordResult, attachSanitized, fail, getQuery.
  emptyState() returns QueryResult.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer ResultSanitizer subscribed to QueryEntity events; on QueryExecuted runs a
  column-scanning redaction pipeline over rawResult.rows: any column named customer_name has
  its value replaced with [REDACTED-NAME]; any column whose string value matches an email
  regex is replaced with [REDACTED-EMAIL]. Computes the list of categories found. Builds
  SanitizedResult, then calls QueryEntity.attachSanitized(sanitized).

- 1 View QueryView with row type QueryRow (mirrors QueryResult minus rawResult.rows — the
  audit log keeps the raw rows; the view holds the sanitized form for the UI). Table updater
  consumes QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question, submittedBy}; mints queryId;
    calls QueryEntity.submit; starts QueryWorkflow; returns {queryId}), GET /queries (list
    from getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: GENERATE_AND_RUN_QUERY =
  Task.name("Generate and run SQL query")
      .description("Translate the natural-language question into SQL, execute it against the receipts table, and return the result rows with a prose summary.")
      .resultConformsTo(QueryResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records QueryRequest, GeneratedSql, ResultRow, RawQueryResult, SanitizedResult,
  QueryResult, QueryStatus.

- SqlGuardrail.java implementing the before-tool-call hook. Receives the tool name and
  arguments map; if tool name is "execute_sql", extracts the "sql" argument and runs the
  four checks listed in eval-matrix.yaml G1. Returns Guardrail.pass() or
  Guardrail.reject(<structured-error-message>).

- src/main/resources/receipts.db — a seeded SQLite file with one table:
    CREATE TABLE receipts (
      receipt_id   TEXT PRIMARY KEY,
      customer_name TEXT,
      customer_email TEXT,
      merchant     TEXT,
      amount_cents INTEGER,
      currency     TEXT,
      receipt_date TEXT,   -- ISO-8601 date
      category     TEXT
    );
  Seeded with 50 rows covering 10 merchants, 8 customers, 4 categories (Food & Beverage,
  Travel, Office Supplies, Software), date range 2025-01-01 to 2026-06-28.

- src/main/resources/schema.sql — the DDL for receipts (above); this is the attachment the
  agent receives so it knows column names without a live DB introspection call.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9505 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The SqlGeneratorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-questions.jsonl with 6 seeded questions:
  "What is the total amount spent per merchant?",
  "Which customer spent the most in the Travel category?",
  "How many receipts were submitted in each month of 2026?",
  "What is the average receipt amount for Office Supplies?",
  "List the 5 most expensive receipts.",
  "Which merchant had the most transactions in Q1 2026?"

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true, decisions.authority_level
  = recommend-only (the agent's answer is informational, not authoritative for financial decisions),
  oversight.human_in_loop = true, failure.failure_modes including "sql-injection-via-llm",
  "hallucinated-row", "pii-leakage-in-result", "schema-hallucination",
  "destructive-query-execution"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/sql-generator.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Text-to-SQL Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with original question, generated
  SQL block with syntax highlighting, guardrail status, sanitized result table, and prose
  summary). Browser title exactly: <title>Akka Sample: Text-to-SQL Agent</title>.
  No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-and-run-sql-query.json — 8 QueryResult entries covering various question types.
      Each entry has a generatedSql block (valid SELECT statement against the receipts schema),
      a rows list (3–8 rows with realistic merchant names, amounts, categories), and a one-
      sentence summary. Plus 2 deliberately MALFORMED entries: one whose generatedSql begins
      with "DELETE FROM receipts" (triggers the G1 guardrail) and one containing "' OR 1=1 --"
      (injection signature). The mock selects a malformed entry on the FIRST iteration of
      every 3rd query (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SqlGeneratorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (generateStep
  90s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the QueryResult row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: QueryTasks.java with GENERATE_AND_RUN_QUERY = Task.name(...).description(...)
  .resultConformsTo(QueryResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9505 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (SqlGeneratorAgent).
  The ResultSanitizer is a Consumer (not an agent) and does not make LLM calls.
- The schema is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated generateStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  bound specifically to tool name "execute_sql". It fires before the tool method executes,
  not after.
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
