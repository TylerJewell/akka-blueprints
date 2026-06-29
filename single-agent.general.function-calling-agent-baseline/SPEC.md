# SPEC — function-calling-agent-baseline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** FunctionCallingAgent.
**One-line pitch:** A user submits a natural-language query; one AI agent decides which registered tools to call, invokes them in a loop until it has enough information, and returns a structured final answer — with every tool call validated before it fires and every answer filtered before it returns.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `FunctionCallingAgent` (AutonomousAgent) drives the entire tool-calling loop; surrounding components only manage the run lifecycle and project state for the UI. Two guardrails govern the agent:

- A **before-tool-call guardrail** intercepts each tool invocation the agent proposes before the tool executes. It validates the tool name against the registered registry, checks that required parameters are present and within declared types, and blocks invocations that reference unknown tools or carry out-of-bounds arguments. A blocked invocation returns a structured rejection to the agent loop so the agent can correct and retry.
- A **before-agent-response guardrail** intercepts the agent's proposed final answer before it reaches the caller. It checks that the response is well-formed `AgentAnswer` JSON, that the `answer` field is non-empty, and that the response does not contain any content from the forbidden-output list (e.g., raw API credentials echoed back from a tool). A blocked response causes the agent to revise within its iteration budget.

The blueprint shows that a tool-calling loop needs guardrails at two distinct cut points — before the side effect (tool call) and before the output (final answer) — and that both cuts are independent.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a query into the **Query** textarea (or picks one of three seeded examples — a multi-step arithmetic word problem, a vocabulary question, and a date-calculation question).
2. The user optionally picks which tools to expose for this run via a **Tool set** checkbox group (calculator, dictionary, date-formatter — all enabled by default).
3. The user clicks **Run query**. The UI POSTs to `/api/runs` and receives a `runId`.
4. The run card appears in the live list in `SUBMITTED` state. Within a second it transitions to `RUNNING` as the workflow starts the agent task.
5. Within 5–30 s the agent finishes its loop. The card transitions to `ANSWER_RECORDED`. The answer appears: a plain-text `answer` string and a collapsible **Tool call trace** showing each tool name, arguments, and result in sequence.
6. The run card also shows how many tool-call iterations the agent used and whether any guardrail fired (badge: BLOCKED\_TOOL\_CALL / BLOCKED\_ANSWER / none).
7. The user can submit another query; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AgentRunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AgentRunEntity`, `AgentRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AgentRunEntity` | `EventSourcedEntity` | Per-run lifecycle: submitted → running → answer → failed. Source of truth. | `AgentRunEndpoint`, `AgentRunWorkflow` | `AgentRunView` |
| `AgentRunWorkflow` | `Workflow` | One workflow per run. Steps: `startStep` → `runStep` → `recordStep`. | started by `AgentRunEndpoint` after entity creation | `FunctionCallingAgent`, `AgentRunEntity` |
| `FunctionCallingAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the query and tool registry in the task definition; calls tools in a loop via function-calling; returns `AgentAnswer`. | invoked by `AgentRunWorkflow` | returns answer |
| `ToolCallGuardrail` | guardrail class | Before-tool-call hook: validates tool name and arguments before every tool invocation. | wired to `FunctionCallingAgent` | blocks or allows tool call |
| `AnswerGuardrail` | guardrail class | Before-agent-response hook: validates final `AgentAnswer` structure and content. | wired to `FunctionCallingAgent` | blocks or allows final answer |
| `AgentRunView` | `View` | Read model: one row per run for the UI. | `AgentRunEntity` events | `AgentRunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolDefinition(
    String toolName,
    String description,
    Map<String, String> parameterTypes   // param name → type hint (e.g. "number", "string")
) {}

record RunRequest(
    String runId,
    String query,
    List<ToolDefinition> enabledTools,
    String submittedBy,
    Instant submittedAt
) {}

record ToolCallRecord(
    int iteration,
    String toolName,
    Map<String, Object> arguments,
    String result,                        // serialised tool output
    boolean blocked,                      // true if ToolCallGuardrail fired
    Instant calledAt
) {}

record AgentAnswer(
    String answer,
    List<ToolCallRecord> toolCallTrace,
    int totalIterations,
    GuardrailSummary guardrailSummary,
    Instant answeredAt
) {}

record GuardrailSummary(
    boolean toolCallBlocked,
    boolean answerBlocked,
    int toolCallBlockCount,
    int answerBlockCount
) {}

record AgentRun(
    String runId,
    Optional<RunRequest> request,
    Optional<AgentAnswer> answer,
    AgentRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AgentRunStatus {
    SUBMITTED, RUNNING, ANSWER_RECORDED, FAILED
}
```

Events on `AgentRunEntity`: `RunSubmitted`, `RunStarted`, `AnswerRecorded`, `RunFailed`.

Every nullable lifecycle field on the `AgentRun` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ query, enabledTools: [ToolDefinition], submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Function Calling Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted runs (status pill + guardrail badge + age) and a right pane with the selected run's detail — query text, enabled tools list, final answer, tool-call trace (collapsible per iteration), and guardrail summary.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`, applied inside `ToolCallGuardrail`): intercepts each proposed tool invocation before execution. Validates: (1) tool name is in the run's `enabledTools` list, (2) all required parameters declared in `ToolDefinition.parameterTypes` are present, (3) no parameter value exceeds declared type bounds (e.g., a "number" parameter must parse as a number). On failure, returns a structured rejection naming the failed check so the agent can correct its arguments or choose a different tool. Blocking prevents invalid or hallucinated tool calls from reaching the tool executor.
- **G2 — before-agent-response guardrail** (`guardrail`, `before-agent-response`, applied inside `AnswerGuardrail`): runs on the agent's proposed final `AgentAnswer`. Asserts: (1) the `answer` field is non-empty, (2) the response is parseable as `AgentAnswer`, (3) the answer does not contain raw API-credential-like tokens (a regex matching `Bearer [A-Za-z0-9._-]{20,}` or similar patterns). On failure, returns a structured rejection; the agent revises within its iteration budget.

## 9. Agent prompts

- `FunctionCallingAgent` → `prompts/function-calling-agent.md`. The single decision-making LLM. System prompt instructs it to receive the user's query, use the available tools as needed, and return a concise final `AgentAnswer` once it has enough information to respond accurately.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the arithmetic seed query with all tools enabled; within 30 s the answer appears with a tool-call trace showing at least one calculator invocation.
2. **J2** — The agent proposes a tool call to an unknown tool name on a run; the `before-tool-call` guardrail blocks it; the agent corrects and tries a valid tool; the final answer is well-formed and the UI shows `toolCallBlocked: true` in the guardrail summary.
3. **J3** — On the mock LLM path, a seeded response contains a credential-like token; the `before-agent-response` guardrail blocks it; the agent revises; the cleaned answer reaches the UI.
4. **J4** — A run with no enabled tools results in a direct answer (no tool calls) within one iteration; the tool-call trace is empty.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named function-calling-agent-baseline demonstrating the single-agent ×
general cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact single-agent-general-function-calling-agent-baseline. Java package
io.akka.samples.functioncallingagentagentworkflow. Akka 3.6.0. HTTP port 9787.

Components to wire (exactly):

- 1 AutonomousAgent FunctionCallingAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/function-calling-agent.md>) and
  .capability(TaskAcceptance.of(AgentRunTasks.RUN_QUERY).maxIterationsPerTask(6)). The
  task receives the user query and enabled tool definitions in its instruction text; tool
  execution is handled by the Akka function-calling mechanism. Output: AgentAnswer{answer:
  String, toolCallTrace: List<ToolCallRecord>, totalIterations: int, guardrailSummary:
  GuardrailSummary, answeredAt: Instant}. The agent is configured with two guardrails:
  a before-tool-call guardrail (ToolCallGuardrail) and a before-agent-response guardrail
  (AnswerGuardrail), both registered via the agent's guardrail-configuration block. On
  guardrail rejection the agent loop retries within its 6-iteration budget.

- 1 Workflow AgentRunWorkflow per runId with three steps:
  * startStep — calls AgentRunEntity.markRunning(), then immediately advances to runStep.
    WorkflowSettings.stepTimeout 5s.
  * runStep — calls componentClient.forAutonomousAgent(FunctionCallingAgent.class,
    "agent-" + runId).runSingleTask(
      TaskDef.instructions(formatQueryAndTools(run.request))
    ) — returns a taskId, then forTask(taskId).result(AgentRunTasks.RUN_QUERY) to fetch
    the answer. On success calls AgentRunEntity.recordAnswer(answer). WorkflowSettings
    .stepTimeout 120s with defaultStepRecovery maxRetries(2).failoverTo(AgentRunWorkflow::error).
  * recordStep — no-op transition; forwards to done.
    error step calls AgentRunEntity.fail(reason) and transitions to done.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AgentRunEntity (one per runId). State AgentRun{runId: String,
  request: Optional<RunRequest>, answer: Optional<AgentAnswer>, status: AgentRunStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. AgentRunStatus enum: SUBMITTED,
  RUNNING, ANSWER_RECORDED, FAILED. Events: RunSubmitted{request}, RunStarted{},
  AnswerRecorded{answer}, RunFailed{reason}. Commands: submit, markRunning, recordAnswer,
  fail, getRun. emptyState() returns AgentRun.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state
  and Optional.of(...) inside the event-applier.

- 1 View AgentRunView with row type AgentRunRow (mirrors AgentRun). Table updater
  consumes AgentRunEntity events. ONE query getAllRuns: SELECT * AS runs FROM
  agent_run_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * AgentRunEndpoint at /api with POST /runs (body {query, enabledTools:
    [{toolName, description, parameterTypes}], submittedBy}; mints runId; calls
    AgentRunEntity.submit; starts AgentRunWorkflow; returns {runId}), GET /runs (list
    from getAllRuns, sorted newest-first), GET /runs/{id} (one row), GET /runs/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- AgentRunTasks.java declaring one Task<R> constant: RUN_QUERY = Task.name("Run query")
  .description("Process the user's query using available tools and return an AgentAnswer")
  .resultConformsTo(AgentAnswer.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ToolDefinition, RunRequest, ToolCallRecord, AgentAnswer,
  GuardrailSummary, AgentRun, AgentRunStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Reads the candidate
  tool invocation (tool name + arguments), runs the three checks listed in eval-matrix.yaml
  G1, and either passes the call through or returns Guardrail.reject(<structured-error>)
  to block the tool call and force the agent loop to retry.

- AnswerGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AgentAnswer from the LLM response, runs the three checks listed in eval-matrix.yaml G2,
  and either passes the response through or returns Guardrail.reject(<structured-error>)
  to force the agent loop to revise.

- InProcessToolExecutor.java — pure deterministic logic (no LLM). Implements three tools:
  * calculator: evaluates a simple arithmetic expression string (integer operands, +−×÷).
  * dictionary: looks up a word from a small in-process map of ~30 entries.
  * date-formatter: formats an ISO date string into a human-readable form.
  Each tool method is called by the Akka function-calling mechanism when the agent names it.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9787 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  FunctionCallingAgent.definition() binds the configured provider via the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-queries.jsonl with 3 seeded queries:
  a 3-step arithmetic word problem (uses calculator twice), a vocabulary question (uses
  dictionary once), and a date-calculation question (uses date-formatter once then
  calculator once).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with domain = general, decisions.authority_level =
  authoritative (the agent's answer is the direct output, no human review in the basic
  path), oversight.human_in_loop = false (baseline pattern — deployers may add it),
  failure.failure_modes including "tool-hallucination", "invalid-tool-argument",
  "credential-leak-in-answer", "infinite-tool-loop"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/function-calling-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Function Calling Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of run cards; right = selected-run detail with query text, enabled
  tools list, final answer, collapsible tool-call trace per iteration, and guardrail
  summary). Browser title exactly: <title>Akka Sample: Function Calling Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(runId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    run-query.json — 6 AgentAnswer entries covering a variety of tool-call traces:
      entries with 1, 2, and 3 tool calls respectively; one with an empty trace (direct
      answer); one with a tool call blocked (toolCallBlocked=true in guardrailSummary,
      toolCallBlockCount=1); one with an answer blocked and revised
      (answerBlocked=true, answerBlockCount=1). Plus 1 deliberately MALFORMED entry with
      a credential-like token in the answer field — the AnswerGuardrail blocks it,
      exercising the before-agent-response retry path.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FunctionCallingAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion AgentRunTasks.java
  MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (startStep 5s, runStep 120s,
  recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on AgentRun is Optional<T>.
- Lesson 7: AgentRunTasks.java with RUN_QUERY = Task.name(...).description(...)
  .resultConformsTo(AgentAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9787 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-
  visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index.
- The single-agent invariant: there is exactly ONE AutonomousAgent (FunctionCallingAgent).
  InProcessToolExecutor is plain Java — it does NOT make an LLM call.
- Both guardrails are wired via the agent's guardrail-configuration block, not as external
  checks.
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
