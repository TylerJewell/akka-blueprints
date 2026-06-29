# SPEC — managed-harness-tool-use-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ManagedHarnessToolUseAgent.
**One-line pitch:** A user submits a natural-language operations question; one AI agent answers it by calling a declared set of tools — query_metrics, list_resources, fetch_logs, compute_stats — in any order; the managed runtime runs the tool-calling loop end-to-end, and a `before-tool-invocation` guardrail validates every tool call against the allowlist before dispatch.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `OperationsAgent` (AutonomousAgent) carries the entire decision; surrounding components prepare its input, execute its tool calls under the managed harness, and record its output. One governance mechanism is wired around the agent:

- A **before-tool-invocation guardrail** runs inside the managed runtime before every tool call the agent requests. It checks the tool name against a configured allowlist and optionally validates the call arguments for obviously dangerous patterns (empty namespace, wildcard selectors). A disallowed tool call is rejected with a structured error returned to the agent loop so the agent can select an allowed alternative or reformulate its approach.

The blueprint shows that a tool-use agent in a managed harness is not implicitly safe just because the runtime controls the loop. The allowlist check is the explicit boundary: the agent knows its tools, the runtime enforces that only declared tools execute.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question in the **Question** textarea (or picks one of four seeded examples — a latency spike investigation, a resource inventory query, a log-error frequency count, a capacity-planning calculation).
2. The user picks an **agent profile** from a dropdown (ops-read-only, ops-read-write, analytics-only) that maps to a different allowlist. Each profile lists its permitted tools.
3. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `RUNNING` — the agent has started calling tools.
5. Within ~10–60 s, the workflow's `runStep` completes. The card transitions to `ANSWERED`. The answer appears: a prose answer paragraph and a **tool-call trace** — an ordered list of every tool the agent invoked, its arguments summary, and a one-line output summary.
6. The user can submit another question; the live list keeps the history visible.
7. If the agent attempted a disallowed tool call, the trace shows a ⛔ row for that call, the agent's recovery, and the final answer.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → running → answered / failed. Source of truth. | `QueryEndpoint`, `QueryWorkflow` | `QueryView` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `runStep` → `recordStep`. | started by `QueryEndpoint` after entity creation | `OperationsAgent`, `QueryEntity` |
| `OperationsAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question and the agent profile's tool declarations; calls tools via the managed runtime; returns `QueryAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `ToolAllowlistGuardrail` | supporting class | Registered on `OperationsAgent` via the `before-tool-invocation` hook. Checks tool name against allowlist; validates arguments for dangerous patterns. | invoked by managed runtime per tool call | accepts or rejects |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolDeclaration(String toolName, String description, List<String> parameterNames) {}

record AgentProfile(String profileId, String label, List<ToolDeclaration> allowedTools) {}

record QueryRequest(
    String queryId,
    String question,
    String profileId,
    String requestedBy,
    Instant submittedAt
) {}

record ToolInvocation(
    String callId,
    String toolName,
    Map<String, String> arguments,
    String outputSummary,
    boolean blocked,
    Instant invokedAt
) {}

record QueryAnswer(
    String prose,
    List<ToolInvocation> toolTrace,
    int toolCallCount,
    int blockedCallCount,
    Instant answeredAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<QueryAnswer> answer,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, RUNNING, ANSWERED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `AgentStarted`, `AnswerRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question, profileId, requestedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Managed-Harness Tool-Use Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + age + question excerpt) and a right pane with the selected query's detail — question, agent profile badge, prose answer, and the ordered tool-call trace (each row: tool name, arguments summary, output summary, blocked indicator).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-invocation guardrail**: runs inside the managed runtime on every tool call `OperationsAgent` requests. Checks (1) the tool name is in the profile's allowlist, (2) no argument value is a bare wildcard (`*`), (3) no argument value is an empty string where one is required. On any failure returns a structured rejection to the agent loop naming the failed check and the disallowed tool name; the agent may then select an alternative tool from its allowed set. Passing calls are dispatched normally.

## 9. Agent prompts

- `OperationsAgent` → `prompts/operations-agent.md`. The single decision-making LLM. System prompt describes the four available tools, their semantics, and when to use each; instructs the agent to think step-by-step before calling a tool; and to produce a final prose answer after the tool-calling loop concludes.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the latency-spike seed question with the ops-read-only profile; within 60 s the answer appears with a tool-call trace containing at least two entries (one `query_metrics`, one `fetch_logs`).
2. **J2** — The agent requests `run_query` (not in the ops-read-only allowlist); the guardrail blocks it; the trace shows the blocked entry; the agent calls `query_metrics` instead and produces the answer.
3. **J3** — The agent attempts to pass `"*"` as an argument; the guardrail rejects the wildcard; the trace shows the blocked entry; the agent narrows the argument and retries.
4. **J4** — The capacity-planning question requires `compute_stats` and `query_metrics` called in sequence; the trace shows both entries in order; the answer cites the result of each.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named managed-harness-tool-use-agent demonstrating the single-agent × general
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-agentcore-managed-harness. Java package
io.akka.samples.managedharnesstooluseagent. Akka 3.6.0. HTTP port 9669.

Components to wire (exactly):

- 1 AutonomousAgent OperationsAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/operations-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUESTION).maxIterationsPerTask(8)).
  Declares four tools: query_metrics(namespace, metric, window), list_resources(kind, filter),
  fetch_logs(service, level, limit), compute_stats(values, operation). Tool implementations
  are in-process stubs that return deterministic canned data keyed on namespace/service/kind.
  Configured with a before-tool-invocation guardrail (G1 in eval-matrix.yaml) registered via
  the agent's guardrail-configuration block. On guardrail rejection the agent loop is notified
  via a structured error and may select an allowed tool.

- 1 Workflow QueryWorkflow per queryId with two steps:
  * runStep — emits AgentStarted on the entity, then calls
    componentClient.forAutonomousAgent(OperationsAgent.class, "ops-" + queryId)
    .runSingleTask(TaskDef.instructions(formatQuestion(request.question(), profile)))
    returns a taskId, then forTask(taskId).result(ANSWER_QUESTION) to fetch the answer.
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * recordStep — calls QueryEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 10s.
    error step calls QueryEntity.fail(reason), stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, answer: Optional<QueryAnswer>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED, RUNNING,
  ANSWERED, FAILED. Events: QuerySubmitted{request}, AgentStarted{}, AnswerRecorded{answer},
  QueryFailed{reason}. Commands: submit, markRunning, recordAnswer, fail, getQuery.
  emptyState() returns Query.initial("") with all Optional fields as Optional.empty() and
  status = SUBMITTED (Lesson 3). Every Optional<T> field uses Optional.empty() in initial
  state and Optional.of(...) inside the event-applier.

- 1 View QueryView with row type QueryRow (mirrors Query). Table updater consumes QueryEntity
  events. ONE query getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question, profileId, requestedBy};
    mints queryId; calls QueryEntity.submit; starts QueryWorkflow; returns {queryId}),
    GET /queries (list from getAllQueries, sorted newest-first), GET /queries/{id} (one row),
    GET /queries/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question").description("Use the available tools to research the question and return a
  QueryAnswer with a prose answer and a full tool-call trace").resultConformsTo(QueryAnswer.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ToolDeclaration, AgentProfile, QueryRequest, ToolInvocation, QueryAnswer,
  Query, QueryStatus.

- ToolAllowlistGuardrail.java implementing the before-tool-invocation hook. Reads the
  tool name and arguments from the pending tool call. Checks (1) tool name is in the
  profile's allowlist, (2) no argument value equals bare "*", (3) no required argument
  is empty. On any failure returns Guardrail.reject(<structured-error>) naming the failed
  check and the disallowed tool or argument; on pass returns Guardrail.allow().

- AgentProfiles.java — a static registry of the three built-in profiles:
  ops-read-only (query_metrics, list_resources, fetch_logs),
  ops-read-write (query_metrics, list_resources, fetch_logs, compute_stats),
  analytics-only (query_metrics, compute_stats).

- ToolStubs.java — in-process implementations of the four tools. Each method is a pure
  function keyed on its primary argument; it returns a deterministic JSON string so the
  agent loop has real-looking data to reason about. Documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9669 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-questions.jsonl with 4 seeded question entries:
  a latency-spike investigation ("Why is the checkout service p99 latency above 2 s?"),
  a resource inventory query ("List all Deployment resources in the payments namespace."),
  a log-error frequency count ("How many ERROR logs did the auth service emit in the last
  hour?"),
  and a capacity-planning calculation ("Given current CPU usage, how many extra pods does
  the inference service need to stay under 70% utilisation?").

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in Section
  8 of this SPEC.

- risk-survey.yaml at the project root with domain.sector pre-filled to "operations", with
  decisions.authority_level = recommend-only (the agent's answer is advisory), oversight
  .human_in_loop = true, failure.failure_modes including "disallowed-tool-invocation",
  "hallucinated-tool-output", "argument-injection", "runaway-tool-loop",
  "stale-data-in-stub"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/operations-agent.md loaded as the agent system prompt.

- README.md at the project root.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with question, profile badge,
  prose answer, and tool-call trace table).
  Browser title exactly: <title>Akka Sample: Managed-Harness Tool-Use Agent</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. For ANSWER_QUESTION, reads
  src/main/resources/mock-responses/answer-question.json, picks one entry pseudo-randomly
  per call (seedFor(queryId)), and deserialises into QueryAnswer.
- answer-question.json — 8 QueryAnswer entries covering all four seeded question types.
  Each entry has a prose paragraph and a toolTrace with 2–4 ToolInvocation entries using
  realistic tool names, arguments, and outputSummaries. Plus 2 entries where the first
  tool call in the trace has blocked=true and toolName not in the ops-read-only allowlist,
  exercising J2. One additional entry exercises J3 (a wildcard argument that was blocked).
- MockModelProvider.seedFor(queryId) makes per-query selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. OperationsAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (runStep 90s, recordStep 10s,
  error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: QueryTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(QueryAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9669 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (OperationsAgent).
  ToolAllowlistGuardrail is a supporting class, not an agent.
- The before-tool-invocation guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external wrapper. It fires inside the managed runtime loop before
  each tool dispatch.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
