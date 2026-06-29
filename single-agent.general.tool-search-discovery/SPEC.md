# SPEC — tool-search-discovery

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Deferred Tool-Discovery Harness.
**One-line pitch:** A user submits a task query; one AI agent searches the tool catalog on demand (never frontloading the full schema set), loads only the matching tool schemas, passes them through an allowlist guardrail before binding, and returns a structured `TaskResult` with the output and the list of tools actually invoked.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `DiscoveryAgent` (AutonomousAgent) carries the entire execution; the surrounding components maintain the tool catalog and enforce access policy. One governance mechanism is wired around the agent:

- A **before-tool-invocation guardrail** runs inside `ToolAllowlistGuardrail` on every tool call the agent wants to make. It checks each tool's id against the configured allowlist. Tools whose ids are absent are stripped from the binding before the agent can call them, and a structured rejection is returned to the agent loop so it may adapt its plan.

The blueprint shows that on-demand tool discovery — loading schemas only when the agent identifies a need — is viable without sacrificing governance: the allowlist check sits between schema loading and invocation and cannot be bypassed.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a task query into the **Task** textarea (or selects one of three seeded examples — a weather-data lookup, a file-search operation, and a text-transformation pipeline).
2. The user fills in the **Submitted by** field and clicks **Submit task**. The UI POSTs to `/api/tasks` and receives a `taskId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `DISCOVERING` as the workflow starts.
4. Within ~2 s the agent completes its tool search. The card transitions to `READY` and the right-pane detail shows the list of discovered tools with their names and descriptions.
5. Within ~10–30 s (depending on model latency), the card transitions to `EXECUTING` then `COMPLETED`. The result panel shows the agent's output text and the tools-used list (which may differ from the discovered list if the guardrail stripped any).
6. If the agent attempted a tool that is not in the allowlist, the guardrail rejection is visible in the right pane as a `guardedTools` chip list alongside a note that those tools were blocked.
7. The user can submit another task; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TaskEntity`, `TaskView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TaskEntity` | `EventSourcedEntity` | Per-task lifecycle: received → discovering → ready → executing → completed / failed. Source of truth. | `TaskEndpoint`, `DiscoveryWorkflow` | `TaskView` |
| `ToolCatalogConsumer` | `Consumer` | Subscribes to `ToolCatalogUpdated` events; merges new entries into the in-process `ToolRegistry`; emits `CatalogRefreshed`. | `TaskEntity` topic | `ToolRegistry` |
| `DiscoveryWorkflow` | `Workflow` | One workflow per task. Steps: `discoverStep` → `executeStep`. Error step transitions entity to `FAILED`. | started by `TaskEndpoint` after `TaskReceived` | `DiscoveryAgent`, `TaskEntity` |
| `DiscoveryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user query as the task instruction and a JSON snapshot of the catalog as an attachment; calls the built-in `tool_search` function to pick schemas; executes the task with the discovered tools; returns `TaskResult`. The `before-tool-invocation` guardrail sits on every outbound tool call. | invoked by `DiscoveryWorkflow` | returns result |
| `ToolAllowlistGuardrail` | supporting class | Implements the `before-tool-invocation` hook. Checks each candidate tool id against `allowlist.json`. Strips disallowed tools and returns a structured rejection listing which ids were blocked. | bound to `DiscoveryAgent` | — |
| `TaskView` | `View` | Read model: one row per task for the UI. | `TaskEntity` events | `TaskEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolEntry(
    String toolId,
    String name,
    String description,
    String category,
    String schemaJson        // the full JSON Schema string for this tool's parameters
) {}

record DiscoveredTools(
    List<ToolEntry> tools,
    Instant discoveredAt
) {}

record TaskRequest(
    String taskId,
    String userQuery,
    String submittedBy,
    Instant submittedAt
) {}

record TaskResult(
    String output,
    List<String> toolsUsed,
    List<String> guardedTools,  // tool ids the guardrail blocked
    Instant completedAt
) {}

record TaskRecord(
    String taskId,
    Optional<TaskRequest> request,
    Optional<DiscoveredTools> discovered,
    Optional<TaskResult> result,
    TaskStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus {
    RECEIVED, DISCOVERING, READY, EXECUTING, COMPLETED, FAILED
}
```

Events on `TaskEntity`: `TaskReceived`, `ToolsDiscovered`, `ExecutionStarted`, `TaskCompleted`, `TaskFailed`.

Every nullable lifecycle field on the `TaskRecord` state is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tasks` — body `{ userQuery, submittedBy }` → `{ taskId }`.
- `GET /api/tasks` — list all tasks, newest-first.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Deferred Tool-Discovery Harness</title>`.

The App UI tab is a two-column layout: a left column with the submission panel and the live list of submitted tasks (status pill + age), and a right pane with the selected task's detail — discovered tools list, guarded-tools chip list, result output, and tools-used list.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-invocation guardrail**: runs on every tool call `DiscoveryAgent` wants to make. Reads `allowlist.json` from classpath, checks the candidate tool's id. If the id is absent, strips the tool and returns a structured rejection to the agent loop, listing the blocked id and the reason (`not-in-allowlist`). The agent may revise its plan; it cannot retry with the same blocked tool within the same task. The `guardedTools` field on the returned `TaskResult` records which ids were blocked so the operator can audit them.

## 9. Agent prompts

- `DiscoveryAgent` → `prompts/discovery-agent.md`. The single decision-making LLM. System prompt instructs it to read the task query, call `tool_search` to find relevant tools, use only the tools whose schemas were returned, and produce a `TaskResult` with its output and the list of tools it actually called.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the weather-data lookup seed; within 30 s the result appears with the weather tool listed in `toolsUsed` and a non-empty `output`.
2. **J2** — The agent's discovered tool list includes a tool whose id is not in the allowlist; the guardrail strips it before invocation; the `TaskResult.guardedTools` field names the blocked id; the task still completes using the remaining tools.
3. **J3** — A query that matches zero catalog entries completes with an empty `toolsUsed` list and an `output` explaining no applicable tools were found; status reaches `COMPLETED`, not `FAILED`.
4. **J4** — A new tool entry is POST-ed to the catalog at runtime; the next task query that would match it discovers and uses it without a service restart.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named tool-search-discovery demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-tool-search-discovery. Java package
io.akka.samples.deferredtooldiscoveryharness. Akka 3.6.0. HTTP port 9532.

Components to wire (exactly):

- 1 AutonomousAgent DiscoveryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/discovery-agent.md>) and
  .capability(TaskAcceptance.of(DISCOVER_AND_EXECUTE).maxIterationsPerTask(5)). The task
  receives the user query as the task instruction text and a JSON snapshot of the current
  tool catalog as a task ATTACHMENT named "catalog.json" (NOT as inline prompt text —
  Akka's TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  TaskResult{output: String, toolsUsed: List<String>, guardedTools: List<String>,
  completedAt: Instant}. The agent is configured with a before-tool-invocation guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.

- 1 Workflow DiscoveryWorkflow per taskId with two steps:
  * discoverStep — calls DiscoveryAgent's built-in tool_search function (via the Akka
    agent-tool contract) with the user query to retrieve matching ToolEntry records from
    ToolRegistry.search(query). Calls TaskEntity.recordDiscovered(discoveredTools). Then
    transitions to executeStep. WorkflowSettings.stepTimeout 15s.
  * executeStep — emits ExecutionStarted, then calls componentClient.forAutonomousAgent(
    DiscoveryAgent.class, "discovery-" + taskId).runSingleTask(
      TaskDef.instructions(request.userQuery())
        .attachment("catalog.json", buildCatalogJson(discovered.tools()).getBytes())
    ) to fetch the result. On success calls TaskEntity.completeTask(result).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(DiscoveryWorkflow::error).
  error step transitions the entity to FAILED. WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TaskEntity (one per taskId). State TaskRecord{taskId: String,
  request: Optional<TaskRequest>, discovered: Optional<DiscoveredTools>,
  result: Optional<TaskResult>, status: TaskStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. TaskStatus enum: RECEIVED, DISCOVERING, READY, EXECUTING,
  COMPLETED, FAILED. Events: TaskReceived{request}, ToolsDiscovered{discovered},
  ExecutionStarted{}, TaskCompleted{result}, TaskFailed{reason}.
  Commands: submit, recordDiscovered, markExecuting, completeTask, fail, getTask.
  emptyState() returns TaskRecord.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer ToolCatalogConsumer subscribed to a "tool-catalog" topic; on ToolCatalogUpdated
  events merges the new ToolEntry list into the in-process ToolRegistry singleton and emits
  a CatalogRefreshed log entry. The registry is seeded at startup from
  src/main/resources/sample-events/tool-catalog.jsonl.

- 1 View TaskView with row type TaskRow (mirrors TaskRecord minus full schemaJson on each
  ToolEntry — the view stores only toolId + name + description per entry). Table updater
  consumes TaskEntity events. ONE query getAllTasks: SELECT * AS tasks FROM task_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks (body {userQuery, submittedBy}; mints taskId;
    calls TaskEntity.submit; starts DiscoveryWorkflow; returns {taskId}), GET /tasks
    (list from getAllTasks, sorted newest-first), GET /tasks/{id} (one row), GET /tasks/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- DiscoveryTasks.java declaring one Task<R> constant: DISCOVER_AND_EXECUTE = Task
  .name("Discover and execute").description("Search the tool catalog for relevant schemas
  and execute the user query, returning a TaskResult").resultConformsTo(TaskResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ToolEntry, DiscoveredTools, TaskRequest, TaskResult, TaskRecord,
  TaskStatus.

- ToolRegistry.java — a thread-safe in-process map of toolId → ToolEntry. search(query)
  returns ToolEntry records whose name or description contains any token from query
  (case-insensitive). Seeded from tool-catalog.jsonl at startup; updated by
  ToolCatalogConsumer at runtime.

- ToolAllowlistGuardrail.java implementing the before-tool-invocation hook. Loads
  allowlist.json from classpath (a JSON array of permitted toolId strings). On each
  candidate tool call it checks the tool id. If not in the allowlist, strips the tool and
  returns Guardrail.reject("tool-not-in-allowlist: " + toolId). Allowed tools pass through.
  The rejection accumulates into the TaskResult.guardedTools list after the task completes.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9532 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/allowlist.json — a JSON array of toolId strings that are permitted.
  Seeded with the ids of the tools in tool-catalog.jsonl except for one entry per
  instruction-set group, which is intentionally absent so J2 is reproducible.

- src/main/resources/sample-events/tool-catalog.jsonl — 8 ToolEntry records covering three
  categories: weather (2 tools: current-weather, forecast-5day), file-ops (3 tools:
  file-search, file-read, file-write), and text (3 tools: text-summarize, text-translate,
  text-classify). file-write is intentionally absent from allowlist.json so the guardrail
  has work to do on file-operation tasks.

- src/main/resources/sample-events/seed-tasks.jsonl — 3 seed task queries: "Get the
  current weather and 5-day forecast for London", "Search for files matching *.log and
  summarize the most recent one", "Translate the following text to French and classify its
  sentiment".

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching Section 8.
  No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = full-automation (the agent acts, does not merely recommend),
  oversight.human_in_loop = false, oversight.human_on_loop = true, failure.failure_modes
  including "tool-invocation-outside-allowlist", "hallucinated-tool-id",
  "schema-mismatch-at-runtime", "catalog-staleness"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/discovery-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Deferred Tool-Discovery Harness",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission panel + live list of task cards; right = selected-task detail with discovered
  tools list, guarded-tools chip list, result output, and tools-used list). Browser title
  exactly: <title>Akka Sample: Deferred Tool-Discovery Harness</title>. No subtitle on the
  Overview tab.

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
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(taskId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    discover-and-execute.json — 6 TaskResult entries covering all three seed task types.
      Each entry has a non-empty output string, a toolsUsed list naming 1–3 tool ids from
      the catalog, and a guardedTools list (empty for most; one entry uses "file-write"
      to exercise the guardrail path for J2). Plus 1 entry with toolsUsed containing a
      fabricated tool id not in the catalog — exercises the guardrail hallucinated-tool-id
      path.
- A MockModelProvider.seedFor(taskId) helper makes per-task selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DiscoveryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DiscoveryTasks.java MUST
  exist.
- Lesson 4: every workflow step has an explicit stepTimeout (discoverStep 15s,
  executeStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the TaskRecord is Optional<T>.
- Lesson 7: DiscoveryTasks.java with DISCOVER_AND_EXECUTE = Task.name(...)
  .description(...).resultConformsTo(TaskResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9532 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (DiscoveryAgent). The tool
  catalog search is a deterministic registry lookup — not a second LLM call.
- The catalog snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions.
- The before-tool-invocation guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external post-hoc check.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
