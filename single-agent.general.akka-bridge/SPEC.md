# SPEC — akka-bridge

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Agents.
**One-line pitch:** A user submits a run request carrying a task description and a set of permitted tools; one AI agent executes the task by calling those tools through an adapter that wraps an external agent framework, with every tool invocation intercepted by a `before-tool-call` guardrail before it leaves the Akka runtime.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `BridgeAgent` (AutonomousAgent) drives the entire execution; the surrounding components manage the run lifecycle and enforce tool-call governance. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** intercepts every outbound tool invocation. Before each tool call is forwarded to the `AgentFrameworkAdapter`, `ToolCallGuardrail` validates the call against a per-tool policy: is the tool in the permitted set for this run? does the call's argument schema match the declared tool signature? has the run exceeded its tool-call budget? A blocked call returns a structured rejection to the agent loop so the agent can attempt an alternative; a permitted call proceeds and is recorded in the entity's audit log.

The blueprint shows that wrapping an external agent framework inside Akka is not just a durability concern — structured governance at the tool-call boundary is the mechanism that makes the wrapped execution auditable and controllable.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a task description into the **Task** textarea (or picks one of three seeded examples — a web-search summary task, a data-extraction task, a multi-step calculation task).
2. The user picks a **tool set** from a dropdown (search-tools, data-tools, math-tools) or enters a custom comma-separated list of permitted tool names.
3. The user clicks **Submit run**. The UI POSTs to `/api/runs` and receives a `runId`.
4. The card appears in the live list in `ACCEPTED` state. Within ~1 s, it transitions to `RUNNING` as the workflow's `agentStep` begins.
5. As the agent calls tools, the card detail shows a live tool-call log: each entry carries the tool name, the call arguments (truncated to 200 chars for display), the guardrail verdict (`permitted` or `blocked`), and the tool result (if permitted).
6. When the agent has finished all tool calls and produced a final answer, the card transitions to `COMPLETED`. The result appears as a short text block below the tool-call log.
7. If the guardrail blocks a call that causes the agent to halt, the card transitions to `BLOCKED`. The guardrail rejection reason is shown inline.
8. The user can submit another run; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `RunEntity`, `RunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RunEntity` | `EventSourcedEntity` | Per-run lifecycle: accepted → running → completed / blocked / failed. Tool-call audit log. | `RunEndpoint`, `RunWorkflow` | `RunView` |
| `AgentFrameworkAdapter` | `Consumer` | Subscribes to `ToolCallPermitted` events; executes the tool call in-process (mocked); writes result back via `RunEntity.recordToolResult`. | `RunEntity` events | `RunEntity` |
| `RunWorkflow` | `Workflow` | One workflow per run. Steps: `agentStep` → `completionStep`. | started by `RunEndpoint` after submit | `BridgeAgent`, `RunEntity` |
| `BridgeAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task description as task instructions and the tool set as tool definitions; calls tools via the guardrail-protected tool-call mechanism; returns `RunResult`. | invoked by `RunWorkflow` | returns result |
| `ToolCallGuardrail` | `Guardrail` (before-tool-call) | Intercepts every outbound tool call from `BridgeAgent`. Validates against the permitted-tools list, argument schema, and budget. | wired on `BridgeAgent` | permits or rejects |
| `RunView` | `View` | Read model: one row per run for the UI. | `RunEntity` events | `RunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolDefinition(
    String toolName,
    String description,
    String argSchema      // JSON Schema string for the tool's argument object
) {}

record RunRequest(
    String runId,
    String taskDescription,
    List<ToolDefinition> permittedTools,
    int toolBudget,       // maximum tool calls allowed for this run
    String submittedBy,
    Instant submittedAt
) {}

record ToolCallRecord(
    String callId,
    String toolName,
    String argumentsJson,
    GuardrailVerdict guardrailVerdict,
    Optional<String> rejectionReason,
    Optional<String> resultJson,
    Instant calledAt,
    Optional<Instant> completedAt
) {}
enum GuardrailVerdict { PERMITTED, BLOCKED }

record RunResult(
    String answer,
    int toolCallsUsed,
    Instant completedAt
) {}

record Run(
    String runId,
    Optional<RunRequest> request,
    List<ToolCallRecord> toolCalls,
    Optional<RunResult> result,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus {
    ACCEPTED, RUNNING, COMPLETED, BLOCKED, FAILED
}
```

Events on `RunEntity`: `RunAccepted`, `RunStarted`, `ToolCallRequested`, `ToolCallPermitted`, `ToolCallBlocked`, `ToolResultRecorded`, `RunCompleted`, `RunBlocked`, `RunFailed`.

Every nullable lifecycle field on the `Run` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ taskDescription, permittedTools: [ToolDefinition], toolBudget, submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition or tool-call update.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Agents</title>`.

The App UI tab is a two-column layout: a left rail with submitted runs (status pill + age + task description truncated to 60 chars) and a right pane with the selected run's detail — task description, permitted tools list, live tool-call log (tool name, arguments, guardrail verdict chip, result snippet), final answer block, and a budget-usage bar.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call initiated by `BridgeAgent`. Asserts (1) the requested tool name is in the run's `permittedTools` list, (2) the call's argument JSON conforms to the tool's declared `argSchema`, and (3) the run has not yet exhausted its `toolBudget`. On any failure returns a structured rejection describing which check failed; the agent loop receives the rejection as a tool error and may adapt. Permitted calls are recorded with verdict `PERMITTED`; blocked calls are recorded with verdict `BLOCKED` and the rejection reason.

## 9. Agent prompts

- `BridgeAgent` → `prompts/bridge-agent.md`. The single decision-making LLM. System prompt instructs it to complete the given task using only the tools provided, to adapt its strategy when a tool call is rejected, and to produce a concise `RunResult` when done.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a search-tools run; within 30 s the run completes with a `RunResult`, the tool-call log shows at least one `PERMITTED` entry, and every entry has a non-empty `resultJson`.
2. **J2** — User submits a run with a tool name not in the permitted set; the guardrail blocks the call; the agent receives a structured rejection and either routes around it or halts; the entity never records a `ToolCallPermitted` event for the blocked tool.
3. **J3** — User submits a run with `toolBudget = 2` and a task that would normally require 4 tool calls; after the second permitted call the guardrail blocks the third (budget exhausted); the entity transitions to `BLOCKED`; the UI shows the partial tool-call log and the block reason.
4. **J4** — The audit log endpoint (`GET /api/runs/{id}`) returns every `ToolCallRecord` in order, each with an exact copy of `argumentsJson` as sent to the framework, confirming the guardrail does not alter the payload after recording.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-bridge demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-akka-bridge. Java package io.akka.samples.agents. Akka 3.6.0.
HTTP port 9562.

Components to wire (exactly):

- 1 AutonomousAgent BridgeAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/bridge-agent.md>) and
  .capability(TaskAcceptance.of(RUN_TASK).maxIterationsPerTask(5)). The task receives
  the task description as its instruction text and the list of permitted tool definitions
  as structured tool metadata registered via the tool-registration block on the agent
  definition. Output: RunResult{answer: String, toolCallsUsed: int, completedAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the
  agent loop receives a tool error response.

- 1 Workflow RunWorkflow per runId with two steps:
  * agentStep — emits RunStarted, then calls componentClient.forAutonomousAgent(
    BridgeAgent.class, "bridge-" + runId).runSingleTask(
      TaskDef.instructions(run.request.taskDescription)
    ) — the tool definitions are registered on the agent's definition, not inlined in
    the task. Returns a taskId, then forTask(taskId).result(RUN_TASK) to fetch the
    RunResult. On success calls RunEntity.complete(result). WorkflowSettings.stepTimeout
    120s with defaultStepRecovery maxRetries(1).failoverTo(RunWorkflow::error).
  * completionStep — calls RunEntity.markCompleted(result). WorkflowSettings.stepTimeout
    10s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity RunEntity (one per runId). State Run{runId: String,
  request: Optional<RunRequest>, toolCalls: List<ToolCallRecord>,
  result: Optional<RunResult>, status: RunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RunStatus enum: ACCEPTED, RUNNING, COMPLETED,
  BLOCKED, FAILED. Events: RunAccepted{request}, RunStarted{}, ToolCallRequested{callId,
  toolName, argumentsJson}, ToolCallPermitted{callId}, ToolCallBlocked{callId,
  rejectionReason}, ToolResultRecorded{callId, resultJson}, RunCompleted{result},
  RunBlocked{reason}, RunFailed{reason}. Commands: accept, start, requestToolCall,
  permitToolCall, blockToolCall, recordToolResult, complete, block, fail, getRun.
  emptyState() returns Run.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier. toolCalls field is an empty List in initial state.

- 1 Consumer AgentFrameworkAdapter subscribed to RunEntity events; on ToolCallPermitted
  executes the corresponding tool call in-process (looks up the ToolCallRecord by callId
  from the current Run state, dispatches to a MockToolExecutor that returns a plausible
  JSON result string, then calls RunEntity.recordToolResult(callId, resultJson)).
  MockToolExecutor dispatches by toolName to per-tool handlers that return seeded
  plausible JSON result strings.

- 1 View RunView with row type RunRow (mirrors Run). Table updater consumes RunEntity
  events. ONE query getAllRuns: SELECT * AS runs FROM run_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * RunEndpoint at /api with POST /runs (body {taskDescription, permittedTools:
    [{toolName, description, argSchema}], toolBudget, submittedBy}; mints runId; calls
    RunEntity.accept; starts RunWorkflow; returns {runId}), GET /runs (list from
    getAllRuns, sorted newest-first), GET /runs/{id} (one row), GET /runs/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- RunTasks.java declaring one Task<R> constant: RUN_TASK = Task.name("Run agent task")
  .description("Execute the given task using the permitted tools and return a RunResult")
  .resultConformsTo(RunResult.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ToolDefinition, RunRequest, ToolCallRecord, GuardrailVerdict,
  RunResult, Run, RunStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Reads the pending
  tool call from the hook context (tool name + arguments JSON). Checks (1) tool name
  is in the run's permittedTools list, (2) arguments JSON conforms to the tool's
  argSchema (simple presence-of-required-keys check), (3) run has not exceeded
  toolBudget (count permitted calls in toolCalls list). On any failure returns
  Guardrail.reject(<structured-error>) to block the call and feed the rejection back
  to the agent loop as a tool error. On pass records the call via
  RunEntity.requestToolCall then RunEntity.permitToolCall and returns Guardrail.permit().

- MockToolExecutor.java — dispatches by toolName to in-process handlers returning
  seeded plausible JSON. Handlers: web_search → {"results": ["result1","result2"]},
  extract_data → {"fields": {"key":"value"}}, calculate → {"answer": 42},
  summarize → {"summary": "Brief summary text"}.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9562 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  The BridgeAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/sample-events/tool-sets.jsonl with 3 seeded tool-set definitions:
  a 3-tool search set (web_search, fetch_url, summarize), a 3-tool data set
  (extract_data, validate_schema, transform_record), and a 2-tool math set
  (calculate, format_number).

- src/main/resources/sample-events/seed-tasks.jsonl with 3 seeded task descriptions
  paired to the tool sets above.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (general
  domain; deployer may override), decisions.authority_level = fully-autonomous
  (the agent executes without a human reviewing each step), oversight.human_in_loop =
  false, oversight.human_on_loop = true (operator monitors runs via the audit log),
  failure.failure_modes including "tool-call-hallucination", "budget-overrun",
  "schema-mismatch", "blocked-tool-bypass-attempt"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/bridge-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Agents", prerequisites,
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = live list of run cards with status pill + age + truncated task; right
  = selected-run detail with task description, permitted tools list, live tool-call
  log, final answer block, and budget-usage bar). Browser title exactly:
  <title>Akka Sample: Agents</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json and picks one entry
  pseudo-randomly per call (seedFor(runId)), deserialising into the typed return.
- Per-task mock-response shapes for THIS blueprint:
    run-agent-task.json — 6 RunResult entries covering all three tool sets.
      Each entry has an answer string and a toolCallsUsed integer. Plus 1 entry
      that returns after simulating a blocked tool rejection (answer explains
      the tool was not available).
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BridgeAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RunTasks.java MUST
  exist.
- Lesson 4: every workflow step has an explicit stepTimeout (agentStep 120s,
  completionStep 10s, error 10s).
- Lesson 6: every nullable lifecycle field on the Run row record is Optional<T>.
- Lesson 7: RunTasks.java with RUN_TASK = Task.name(...).description(...)
  .resultConformsTo(RunResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated models.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9562 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND
  the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BridgeAgent).
  MockToolExecutor is plain Java with no LLM call. AgentFrameworkAdapter is a
  Consumer, not an agent.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as a post-call check. The hook fires before the tool call reaches
  AgentFrameworkAdapter.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
