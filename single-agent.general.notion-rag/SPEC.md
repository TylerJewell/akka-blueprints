# SPEC — notion-rag

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** NotionRAG.
**One-line pitch:** A user asks a question; one AI agent queries a bound Notion database, reads the retrieved rows (passed as a task attachment, never as inline prompt text), and returns a grounded `QueryAnswer` — a direct answer, a confidence level, and one `SourceCitation` per supporting row.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `NotionQueryAgent` (AutonomousAgent) carries the entire answer-generation decision; the surrounding components retrieve the relevant Notion rows, pass them as a typed attachment, and validate that the answer is grounded in what was returned. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** (`GroundingGuardrail`) runs on every turn of `NotionQueryAgent`. It checks that every `SourceCitation.rowId` in the candidate answer appears in the set of row IDs that were actually retrieved from Notion, and that at minimum one citation is present. An ungrounded or citation-free answer triggers a retry inside the same task.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — a targeted grounding check sits between the model's response and the caller without requiring a second agent.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** input (or picks one of three seeded questions about a product-catalogue Notion database: "Which products support SSO?", "What is the pricing for the Enterprise tier?", "Which items are marked as deprecated?").
2. The user optionally picks a **session label** (for grouping related questions in the history) or leaves it blank.
3. The user clicks **Ask**. The UI POSTs to `/api/sessions/{id}/questions` and receives a `questionId`.
4. The question appears in the conversation panel in `SUBMITTED` state. Within ~2 s, the retriever runs and the card transitions to `ROWS_RETRIEVED` — the number of matching rows is shown.
5. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWERED`. The answer text appears, followed by a `SourceCitations` table (row ID, property excerpt, match reason).
6. The user may ask a follow-up question in the same session; the history panel accumulates all turns.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create session, submit question, list, get, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: tracks all questions, their retrieved rows, and answers. Source of truth. | `QueryEndpoint`, `NotionRetriever`, `QueryWorkflow` | `SessionView` |
| `NotionRetriever` | `Consumer` | Subscribes to `QuestionSubmitted` events; queries Notion API for matching rows; calls `SessionEntity.attachRows`. | `SessionEntity` events | `SessionEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per question. Steps: `awaitRowsStep` → `answerStep`. | started by `NotionRetriever` once rows land | `NotionQueryAgent`, `SessionEntity` |
| `NotionQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user question as task instructions and the retrieved Notion rows as a task attachment; returns `QueryAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `GroundingGuardrail` | supporting class | `before-agent-response` hook on `NotionQueryAgent`. Validates every cited `rowId` is in the retrieved set and at least one citation exists. | agent turn output | agent loop |
| `SessionView` | `View` | Read model: one row per question for the UI. | `SessionEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record DatabaseSchema(
    String databaseId,
    String databaseName,
    List<PropertyDefinition> properties
) {}

record PropertyDefinition(
    String propertyId,
    String name,
    String type   // "title","rich_text","number","select","multi_select","checkbox","url","date"
) {}

record NotionRow(
    String rowId,
    Map<String, String> properties   // propertyName -> rendered value
) {}

record RetrievedRows(
    String databaseId,
    String queryText,
    List<NotionRow> rows,
    int totalRowsFetched
) {}

record SourceCitation(
    String rowId,
    String propertyName,
    String excerpt,
    String matchReason
) {}

record QueryAnswer(
    String answer,
    AnswerConfidence confidence,
    List<SourceCitation> citations,
    Instant answeredAt
) {}
enum AnswerConfidence { HIGH, MEDIUM, LOW, NO_DATA }

record Question(
    String questionId,
    String sessionId,
    String questionText,
    String submittedBy,
    Instant submittedAt,
    Optional<RetrievedRows> retrievedRows,
    Optional<QueryAnswer> answer,
    QuestionStatus status
) {}

enum QuestionStatus {
    SUBMITTED, ROWS_RETRIEVED, ANSWERING, ANSWERED, FAILED
}

record Session(
    String sessionId,
    String label,
    List<Question> questions,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> lastActivityAt
) {}

enum SessionStatus { ACTIVE, CLOSED }
```

Events on `SessionEntity`: `SessionCreated`, `QuestionSubmitted`, `RowsRetrieved`, `AnswerStarted`, `AnswerRecorded`, `QuestionFailed`.

Every nullable lifecycle field on `Question` and `Session` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ label, createdBy }` → `{ sessionId }`.
- `POST /api/sessions/{sessionId}/questions` — body `{ questionText, submittedBy }` → `{ questionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{sessionId}` — one session with all questions.
- `GET /api/sessions/{sessionId}/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: NotionRAG</title>`.

The App UI tab is a two-column layout: a left rail with the session list (session label, question count, last-activity age) and a right pane with the active session's conversation — each turn shows the question text, the row-retrieval summary, the answer, and the citation table.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `NotionQueryAgent`. Asserts the candidate response is a parseable `QueryAnswer` JSON, every `citations[].rowId` is present in the `RetrievedRows.rows` list passed to the task, and the `citations` list is non-empty (a zero-citation answer is hallucination risk). On any failure, returns a structured rejection naming the failed check; the agent loop retries within its 3-iteration budget. If all 3 iterations fail, the workflow transitions the question to `FAILED`.

## 9. Agent prompts

- `NotionQueryAgent` → `prompts/notion-query-agent.md`. The single decision-making LLM. System prompt instructs it to read the retrieved rows attachment, answer the question directly, and back every factual claim with a `SourceCitation` referencing an actual row ID and property.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks "Which products support SSO?" against the seeded product-catalogue schema; within 30 s the answer appears with at least one `SourceCitation` whose `rowId` is in the retrieved set, and the grounding guardrail emits no rejections.
2. **J2** — With the mock LLM, every third question submission produces a candidate answer containing a rowId not present in the retrieved set; the `before-agent-response` guardrail rejects it; the second iteration produces a grounded answer; the UI never displays the ungrounded response.
3. **J3** — A question that matches zero Notion rows produces an answer with `confidence = NO_DATA` and an empty citations list whose guardrail pass is handled by a special zero-row exemption path.
4. **J4** — Follow-up question in the same session appends correctly; the session history in the UI shows both turns in order.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named notion-rag demonstrating the single-agent × general cell. Maven group
io.akka.samples. Maven artifact single-agent-general-notion-rag. Java package
io.akka.samples.notionaiassistantgenerator. Akka 3.6.0. HTTP port 9756.

Components to wire (exactly):

- 1 AutonomousAgent NotionQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/notion-query-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUESTION).maxIterationsPerTask(3)). The task receives
  the user's question in the task instructions field and the retrieved Notion rows JSON as a
  task ATTACHMENT named "rows.json" (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  QueryAnswer{answer: String, confidence: AnswerConfidence (HIGH/MEDIUM/LOW/NO_DATA),
  citations: List<SourceCitation>, answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow QueryWorkflow per questionId with two steps:
  * awaitRowsStep — polls SessionEntity.getSession every 1s looking for the question whose
    status == ROWS_RETRIEVED; on found advances to answerStep. WorkflowSettings.stepTimeout
    15s.
  * answerStep — emits AnswerStarted, then calls componentClient.forAutonomousAgent(
    NotionQueryAgent.class, "agent-" + questionId).runSingleTask(
      TaskDef.instructions("Question: " + question.questionText())
        .attachment("rows.json", serializeRows(retrievedRows).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANSWER_QUESTION) to fetch the answer.
    On success calls SessionEntity.recordAnswer(questionId, answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error). error step transitions the question to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  label: String, questions: List<Question>, status: SessionStatus, createdAt: Instant,
  lastActivityAt: Optional<Instant>}. SessionStatus enum: ACTIVE, CLOSED. QuestionStatus
  enum: SUBMITTED, ROWS_RETRIEVED, ANSWERING, ANSWERED, FAILED. Events: SessionCreated{label,
  createdBy}, QuestionSubmitted{questionId, questionText, submittedBy, submittedAt},
  RowsRetrieved{questionId, retrievedRows}, AnswerStarted{questionId},
  AnswerRecorded{questionId, answer}, QuestionFailed{questionId, reason}. Commands:
  createSession, submitQuestion, attachRows, markAnswering, recordAnswer, failQuestion,
  getSession, closeSession. emptyState() returns Session.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer NotionRetriever subscribed to SessionEntity events; on QuestionSubmitted:
  (1) reads the bound Notion database ID from application.conf (notion.database-id), (2) calls
  the Notion search API (POST https://api.notion.com/v1/databases/{id}/query with the
  question text as a filter), (3) maps the API response to List<NotionRow>, (4) builds
  RetrievedRows, then calls SessionEntity.attachRows(questionId, retrievedRows). After
  attachRows lands, starts a QueryWorkflow with id = "query-" + questionId. The Notion API
  token is read from the configured env var ${?NOTION_API_KEY} only — never stored on any
  entity or logged.

- 1 View SessionView with row type SessionRow (mirrors Question minus internal workflow ids).
  Table updater consumes SessionEntity events. ONE query getAllSessions: SELECT * AS sessions
  FROM session_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /sessions (body {label, createdBy}; mints sessionId;
    calls SessionEntity.createSession; returns {sessionId}), POST /sessions/{sessionId}/
    questions (body {questionText, submittedBy}; mints questionId; calls
    SessionEntity.submitQuestion; returns {questionId}), GET /sessions (list from
    getAllSessions, sorted newest-first), GET /sessions/{sessionId} (one session), GET
    /sessions/{sessionId}/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question").description("Read the attached Notion rows and produce a QueryAnswer for the
  submitted question").resultConformsTo(QueryAnswer.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records DatabaseSchema, PropertyDefinition, NotionRow, RetrievedRows, SourceCitation,
  QueryAnswer, AnswerConfidence, Question, QuestionStatus, Session, SessionStatus.

- GroundingGuardrail.java implementing the before-agent-response hook. Reads the candidate
  QueryAnswer from the LLM response, extracts the set of rowIds from the retrieved rows
  attachment (passed via the task context), runs two checks: (1) every citations[].rowId
  appears in the retrieved set; (2) if retrievedRows is non-empty, citations is non-empty.
  Zero-row exemption: if retrievedRows.rows is empty, a NO_DATA answer with empty citations
  passes without error. On any failure returns Guardrail.reject(<structured-error>) to force
  the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9756 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also notion.database-id =
  ${?NOTION_DATABASE_ID} (default "SEEDED_DEMO_DB" for mock mode). The NotionQueryAgent
  .definition() binds the configured provider via .modelProvider("${akka.javasdk.agent
  .default}") or the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/schema-snapshot.jsonl — a seeded product-catalogue Notion
  database schema: 8 properties (Name/title, Category/select, PricingTier/select,
  SupportsSSO/checkbox, Status/select, DocumentationURL/url, LastUpdated/date,
  Description/rich_text) plus 12 sample rows covering diverse combinations. Each row
  contains realistic fictional product/feature names. Three rows contain the literal
  checkbox value true for SupportsSSO so J1 can be verified.

- src/main/resources/sample-events/seed-questions.jsonl — three seeded question strings:
  "Which products support SSO?", "What is the pricing for the Enterprise tier?",
  "Which items are marked as deprecated?".

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false (Notion rows in
  the seeded schema contain no PII), decisions.authority_level = recommend-only (the agent's
  answer is informational, not authoritative), oversight.human_in_loop = false (lookup Q&A
  is non-critical), failure.failure_modes including "hallucinated-row", "missing-citation",
  "stale-schema", "ungrounded-answer"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/notion-query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: NotionRAG", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  session list + active-session conversation; right = selected-question detail with
  retrieved-row count, answer text, and citation table).
  Browser title exactly: <title>Akka Sample: NotionRAG</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- /akka:specify must also inspect for NOTION_API_KEY and NOTION_DATABASE_ID. If both are
  set, record them in application.conf and proceed. If only NOTION_API_KEY is set, ask the
  user for the database ID. If neither is set, offer mock mode (the NotionRetriever returns
  the seeded rows from schema-snapshot.jsonl instead of calling the live API).
- If no model-provider key is set, ask the user how to source the key, offering five
  options: (a) Mock LLM, (b) name an existing env var, (c) env file path, (d) secrets-store
  URI, (e) type once in this session. See AKKA-EXEMPLAR-LESSONS.md Lesson 25.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(questionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 8 QueryAnswer entries. Each entry cites rowIds drawn from the
      seeded schema-snapshot.jsonl (rows 1–12). Confidence values vary: 4 HIGH, 2 MEDIUM,
      1 LOW, 1 NO_DATA. Plus 2 deliberately MALFORMED entries (one with a citation rowId
      not present in the seeded rows; one with an empty citations list despite rows being
      non-empty) — the guardrail blocks both, exercising the retry path. The mock selects
      a malformed entry on the FIRST iteration of every 3rd question (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(questionId) helper makes per-question selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. NotionQueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitRowsStep 15s, answerStep
  60s, error 5s).
- Lesson 6: every nullable lifecycle field on Question and Session uses Optional<T>.
- Lesson 7: QueryTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(QueryAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9756 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs with a Notion connection or in mock mode" — never
  T1/T2/T3/T4 in any user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel, NEVER by NodeList index. Exactly five
  <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (NotionQueryAgent). The
  retrieval step (NotionRetriever) is a Consumer — it calls an API, not an LLM. The grounding
  check (GroundingGuardrail) is deterministic logic — not an LLM. Exactly one component talks
  to a model.
- The retrieved rows are passed as a Task ATTACHMENT ("rows.json"), never inlined into the
  agent's instructions. Verify the generated answerStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism. See Lesson 1's AutonomousAgent contract for how the hook is registered.
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
