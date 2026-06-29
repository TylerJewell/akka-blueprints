# SPEC — json-query-engine

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** JSON Query Engine Workflow.
**One-line pitch:** A user submits a natural-language question and a document identifier; one `QueryAgent` walks it through three task phases — **PARSE** the question into a traversal plan, **TRAVERSE** the JSON document using generated path expressions, **RESPOND** with a grounded natural-language answer — with each phase gated on the prior phase's recorded output and each phase's path-expression tools blocked when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a general-purpose JSON-querying domain. One `QueryAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PARSE task's typed output (a traversal plan) becomes the TRAVERSE task's instruction context; the TRAVERSE task's typed result (a set of matched path/value pairs) becomes the RESPOND task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It validates generated path expressions before they are executed against the JSON document store. A call to `resolvePathExpression` or `extractValueAt` is rejected if the path expression is structurally malformed (fails JSON-path syntax), references a document root key that does not exist in the document's `DocumentSchema`, or declares a traversal depth exceeding the configured limit. The rejection returns a structured error to the agent so the task loop can correct the path inside its iteration budget. The same hook enforces the phase-boundary property: a RESPOND-phase tool (`formatAnswer`) is unreachable while the entity has not yet recorded `PathsTraversed`.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and per-phase tool isolation. Path validation at the tool boundary catches LLM-generated path errors before they touch data, surfacing them as structured rejections rather than runtime exceptions.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **question** into the input (e.g., "What is the price of the Pro plan?") and selects a **document** from the dropdown (or picks one of three seeded documents — `saas-pricing-catalog`, `team-directory`, `infrastructure-config`).
2. The user clicks **Run query**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the agent has been handed the PARSE task.
4. Within ~10–20 s the card reaches `TRAVERSING` — the `ParsedQuestion` is visible in the card detail (a traversal plan showing the identified query intent, entity type, and candidate root keys). The agent's PARSE task returned; the workflow recorded `QuestionParsed` and ran the TRAVERSE task.
5. Within ~10–20 s more the card reaches `RESPONDING`. The `TraversalResult` is visible (a table of matched paths with their extracted values and the JSON depth at which each was found).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full `QueryResult` — the natural-language answer, a citations list linking each claim to a matched path and value, and an accuracy score chip (1–5) with a one-line rationale.
7. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: created → parsing → parsed → traversing → traversed → responding → responded → evaluated. Source of truth. | `QueryEndpoint`, `JsonQueryWorkflow` | `QueryView` |
| `JsonQueryWorkflow` | `Workflow` | One workflow per query. Steps: `parseStep` → `traverseStep` → `respondStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `QueryEndpoint` after `CREATED` | `QueryAgent`, `QueryEntity` |
| `QueryAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `QueryTasks.java`: `PARSE_QUESTION` → `ParsedQuestion`, `TRAVERSE_DOCUMENT` → `TraversalResult`, `COMPOSE_RESPONSE` → `QueryResult`. Each task is registered with the phase-appropriate function tools. | invoked by `JsonQueryWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `identifyQueryIntent(question)` and `selectRootKeys(schema, intent)`. Reads from `src/main/resources/sample-data/schemas/<docId>.json` for schema metadata. | called from PARSE task | returns `QueryIntent` / `List<String>` |
| `TraverseTools` | function-tools class | Implements `resolvePathExpression(docId, pathExpr)` and `extractValueAt(docId, path)`. Reads from `src/main/resources/sample-data/documents/<docId>.json`. | called from TRAVERSE task | returns `List<PathMatch>` / `JsonValue` |
| `RespondTools` | function-tools class | Implements `formatAnswer(intent, matches)` and `buildCitations(matches)`. Pure in-memory transformations over the traversal result. | called from RESPOND task | returns `String` / `List<Citation>` |
| `PathGuardrail` | `before-tool-call` guardrail (registered on `QueryAgent`) | Reads the in-flight task's declared phase and the current `QueryEntity` status. Validates path expressions structurally before any document read. Rejects malformed paths, unknown root keys, and out-of-phase tool calls. | every tool call on every task | accept / structured-reject |
| `AccuracyScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `QueryResult`, `TraversalResult`, `ParsedQuestion`. Output: `AccuracyResult{score, rationale}`. | called from `evalStep` | returns score |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record QueryIntent(String intentType, String targetEntity, List<String> candidateRootKeys) {}

record ParsedQuestion(String questionText, QueryIntent intent, Instant parsedAt) {}

record PathMatch(String pathExpression, String resolvedValue, int depthLevel) {}

record TraversalResult(List<PathMatch> matches, String docId, Instant traversedAt) {}

record Citation(String pathExpression, String value, String label) {}

record QueryResult(
    String answer,
    List<Citation> citations,
    Instant composedAt
) {}

record AccuracyResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record QueryRecord(
    String queryId,
    Optional<String> question,
    Optional<String> docId,
    Optional<ParsedQuestion> parsed,
    Optional<TraversalResult> traversal,
    Optional<QueryResult> result,
    Optional<AccuracyResult> accuracy,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    CREATED, PARSING, PARSED, TRAVERSING, TRAVERSED,
    RESPONDING, RESPONDED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QueryCreated`, `ParseStarted`, `QuestionParsed`, `TraverseStarted`, `PathsTraversed`, `RespondStarted`, `ResponseComposed`, `AccuracyScored`, `GuardrailRejected`, `QueryFailed`.

Every nullable lifecycle field on the `QueryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question, docId }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: JSON Query Engine Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of queries (status pill + question + age) and a right pane with the selected query's detail — question, document id, parsed traversal plan, matched path/value pairs table, composed answer with citations, accuracy score chip, and a path-rejection log strip if any guardrail rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (path-validation gate)**: `PathGuardrail` is registered on `QueryAgent` and runs before every tool call. For path-expression tools (`resolvePathExpression`, `extractValueAt`), it validates the expression against three rules: (1) syntax check — the path expression must be a well-formed JSON-path expression (starts with `$`, only uses `.`, `[]`, `*`, `..` operators); (2) root-key check — the first key segment after `$` must appear in the document's `DocumentSchema.rootKeys`; (3) depth check — the path nesting depth must not exceed the schema's `maxTraversalDepth`. For phase-boundary enforcement, TRAVERSE tools require `status ∈ {PARSED, TRAVERSING}` AND `parsed.isPresent()`; RESPOND tools require `status ∈ {TRAVERSED, RESPONDING}` AND `traversal.isPresent()`. On reject, the guardrail returns a structured `path-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, pathExpr, reason}` event for visibility. The agent loop retries within its 4-iteration budget.
- **E1 — `on-decision-eval`**: runs immediately after `ResponseComposed` lands, as `evalStep` inside the workflow. `AccuracyScorer` is deterministic and rule-based (no LLM call). Four checks, one point per check satisfied, on a base of 1: (1) path coverage — every candidate root key from `ParsedQuestion.intent.candidateRootKeys` has at least one matching `PathMatch` in the `TraversalResult`; (2) answer grounding — every `Citation.value` in the `QueryResult.citations` list appears verbatim in `TraversalResult.matches[].resolvedValue`; (3) citation completeness — every `PathMatch` in `TraversalResult.matches` that was used in the answer has a corresponding `Citation`; (4) intent alignment — the `QueryResult.answer` is non-empty and the answer's length in characters is at least 20 (guards against stub responses). Emits `AccuracyScored{score:1..5, rationale}` on a one-point-per-rule basis.

## 9. Agent prompts

- `QueryAgent` → `prompts/query-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded question "What is the price of the Pro plan?" against `saas-pricing-catalog`; within 60 s the query reaches `EVALUATED` with a non-empty answer, ≥ 1 citation, and an accuracy score chip showing ≥ 4.
2. **J2** — The agent generates a malformed path expression (mock LLM path, expression does not start with `$`). `PathGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries with a corrected path; the query eventually completes. The UI's path-rejection log strip shows the one rejected call.
3. **J3** — A query whose mock-LLM RESPOND task cites a value absent from the recorded `TraversalResult` is scored 1 with a rationale naming the ungrounded citation; the UI flags the card.
4. **J4** — Each task's tool calls are visible in the per-query trace (logged at `INFO`); the PARSE task's log shows only PARSE-tool calls, the TRAVERSE task's log shows only TRAVERSE-tool calls, the RESPOND task's log shows only RESPOND-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named json-query-engine demonstrating the sequential-pipeline x general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-json-query-engine. Java package
io.akka.samples.jsonqueryengineworkflow. Akka 3.6.0. HTTP port 9228.

Components to wire (exactly):

- 1 AutonomousAgent QueryAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/query-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  PARSE, TRAVERSE, and RESPOND tool sets are ALL registered on the agent; phase gating and
  path validation are the job of PathGuardrail, NOT of conditional .tools(...) wiring. The
  before-tool-call guardrail (PathGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow JsonQueryWorkflow per queryId with four steps:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(QueryAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions("Question: " + question + "\nDocument: " + docId
        + "\nPhase: PARSE\nLoad the document schema and identify the query intent,
        target entity, and candidate root keys.")
        .metadata("queryId", queryId)
        .metadata("phase", "PARSE")
        .taskType(QueryTasks.PARSE_QUESTION)
    ). Reads forTask(taskId).result(PARSE_QUESTION) to get ParsedQuestion. Writes
    QueryEntity.recordParsed(parsed). WorkflowSettings.stepTimeout 60s.
  * traverseStep — emits TraverseStarted, then runSingleTask with TaskDef.instructions
    (formatTraverseContext(parsed, question, docId)) and metadata.phase = "TRAVERSE", taskType
    TRAVERSE_DOCUMENT. Writes QueryEntity.recordTraversal(traversal). stepTimeout 60s.
  * respondStep — emits RespondStarted, then runSingleTask with TaskDef.instructions
    (formatRespondContext(traversal, parsed, question)) and metadata.phase = "RESPOND", taskType
    COMPOSE_RESPONSE. Writes QueryEntity.recordResult(result). stepTimeout 60s.
  * evalStep — runs the deterministic AccuracyScorer over (result, traversal, parsed) and
    writes QueryEntity.recordAccuracy(accuracy). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(JsonQueryWorkflow::error). The error step writes QueryFailed
  and ends.

- 1 EventSourcedEntity QueryEntity (one per queryId). State QueryRecord{queryId,
  question: Optional<String>, docId: Optional<String>, parsed: Optional<ParsedQuestion>,
  traversal: Optional<TraversalResult>, result: Optional<QueryResult>,
  accuracy: Optional<AccuracyResult>, status: QueryStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. QueryStatus enum: CREATED, PARSING, PARSED, TRAVERSING,
  TRAVERSED, RESPONDING, RESPONDED, EVALUATED, FAILED. Events:
  QueryCreated{question, docId}, ParseStarted, QuestionParsed{parsed},
  TraverseStarted, PathsTraversed{traversal}, RespondStarted, ResponseComposed{result},
  AccuracyScored{accuracy}, GuardrailRejected{phase, tool, pathExpr, reason}, QueryFailed{reason}.
  Commands: create, startParse, recordParsed, startTraverse, recordTraversal, startRespond,
  recordResult, recordAccuracy, recordGuardrailRejection, fail, getQuery. emptyState()
  returns QueryRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View QueryView with row type QueryRow that mirrors QueryRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes QueryEntity events. ONE query
  getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question, docId}; mints queryId; calls
    QueryEntity.create(question, docId); then starts JsonQueryWorkflow with id
    "workflow-" + queryId; returns {queryId}), GET /queries (list from getAllQueries, sorted
    newest-first), GET /queries/{id} (one row), GET /queries/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- QueryTasks.java declaring three Task<R> constants:
    PARSE_QUESTION = Task.name("Parse question").description("Identify query intent, target
      entity, and candidate root keys from the question and document schema")
      .resultConformsTo(ParsedQuestion.class);
    TRAVERSE_DOCUMENT = Task.name("Traverse document").description("Resolve path expressions
      against the JSON document to extract matching values")
      .resultConformsTo(TraversalResult.class);
    COMPOSE_RESPONSE = Task.name("Compose response").description("Write a grounded
      natural-language answer citing matched path/value pairs from the traversal result")
      .resultConformsTo(QueryResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {PARSE, TRAVERSE, RESPOND}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "identifyQueryIntent", phase = Phase.PARSE)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- ParseTools.java — @FunctionTool identifyQueryIntent(String question) -> QueryIntent
  parsing the question for intent type, target entity, and candidate root keys;
  @FunctionTool selectRootKeys(String docId, QueryIntent intent) -> List<String> reading
  the document schema from src/main/resources/sample-data/schemas/<docId>.json and returning
  only the root keys that match the intent's candidateRootKeys.

- TraverseTools.java — @FunctionTool resolvePathExpression(String docId, String pathExpr)
  -> List<PathMatch> reading src/main/resources/sample-data/documents/<docId>.json and
  evaluating the path expression, returning one PathMatch per hit (path, value, depth);
  @FunctionTool extractValueAt(String docId, String path) -> JsonValue reading the exact
  path from the document and returning the scalar or object value at that location.

- RespondTools.java — @FunctionTool formatAnswer(QueryIntent intent, List<PathMatch> matches)
  -> String composing a natural-language answer sentence from the intent and matched values;
  @FunctionTool buildCitations(List<PathMatch> matches) -> List<Citation> building one
  Citation per PathMatch (pathExpression, value, label derived from the last path segment).

- PathGuardrail.java — implements the before-tool-call hook. For path-expression tools
  (resolvePathExpression, extractValueAt), validates: (1) expression starts with "$" and uses
  only valid JSON-path operators; (2) first key segment after "$" is in the document's schema
  rootKeys; (3) path nesting depth (count of "." and "[]" delimiters) does not exceed schema's
  maxTraversalDepth. For all tools, checks phase-status matrix (PARSE tools: {CREATED,PARSING};
  TRAVERSE tools: {PARSED,TRAVERSING} + parsed.isPresent(); RESPOND tools:
  {TRAVERSED,RESPONDING} + traversal.isPresent()). On reject calls
  QueryEntity.recordGuardrailRejection and returns Guardrail.reject("path-violation: <reason>").

- AccuracyScorer.java — pure deterministic logic (no LLM). Inputs: QueryResult, TraversalResult,
  ParsedQuestion. Outputs: AccuracyResult with score and rationale. Four checks, one point each
  from a base of 1: path coverage (every candidateRootKey has ≥1 PathMatch), answer grounding
  (every Citation.value in QueryResult.citations verbatim in TraversalResult matches),
  citation completeness (every PathMatch used in answer has a Citation), intent alignment
  (answer non-empty and ≥ 20 chars). Score range 1-5.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9228 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/documents/ — three JSON files: saas-pricing-catalog.json
  (nested pricing tiers and feature flags), team-directory.json (department hierarchy with
  members and roles), infrastructure-config.json (service topology with hosts, ports, and
  environment configs). Each has 3-4 top-level keys and 3-5 levels of nesting.

- src/main/resources/sample-data/schemas/ — three matching schema files <docId>.json each
  containing {rootKeys: [...], maxTraversalDepth: 5, docId: "..."} so PathGuardrail can
  validate root-key and depth constraints.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (the JSON
  documents are configuration/product data, not person-level data), decisions.authority_level
  = recommend-only (the answer is advisory), oversight.human_in_loop = true (a human reads
  the answer before acting on it), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "malformed-path-expression",
  "ungrounded-citation", "phase-violation", "empty-traversal"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: JSON Query Engine Workflow",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with question header, parsed
  traversal plan, path/value matches table, composed answer with citations, accuracy score
  chip, path-rejection log strip). Browser title exactly:
  <title>Akka Sample: JSON Query Engine Workflow</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
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
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    parse-question.json — 6 ParsedQuestion entries, each with a QueryIntent identifying
      intent type (lookup/filter/aggregate), target entity, and 2-3 candidateRootKeys drawn
      from the seeded documents. Each entry's tool_calls array: 1 identifyQueryIntent call +
      1 selectRootKeys call. Plus 1 entry whose tool_calls includes a resolvePathExpression
      call (a TRAVERSE-phase tool called during the PARSE phase) — the guardrail rejects it;
      the mock then falls through to a normal parse sequence. Select the violating entry on
      the FIRST iteration of every 3rd query (modulo seed) so J2 is reproducible.
    traverse-document.json — 6 TraversalResult entries each with 2-5 PathMatch items
      referencing real paths in the seeded documents. tool_calls: 1-2 resolvePathExpression
      calls + 1-2 extractValueAt calls.
    compose-response.json — 6 QueryResult entries each with a non-empty answer and 1-3
      Citations referencing matched path values. Plus 1 entry whose Citation.value does not
      appear in the paired TraversalResult — the evalStep scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. QueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, traverseStep 60s, respondStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on QueryRecord is Optional<T>. The view table
  updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: QueryTasks.java with PARSE_QUESTION, TRAVERSE_DOCUMENT, COMPOSE_RESPONSE
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9228 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative — shape, minimal, smaller, complex, Akka SDK
  in prose, marketing tone, competitor brand names.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (QueryAgent). The
  on-decision eval is rule-based (AccuracyScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (PathGuardrail) is the runtime mechanism that enforces
  the phase order. Do NOT conditionally register tools per task — the guardrail is the gate.
- Task dependency is carried by typed task results: parseStep writes ParsedQuestion onto
  the entity, traverseStep reads it and builds the TRAVERSE task's instruction context from
  it, respondStep reads both. The agent itself is stateless across phases.
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
