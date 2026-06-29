# SPEC — akka-resumable-agent-http

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** DurableAgentHTTP.
**One-line pitch:** A user POSTs a location query to `/agent/run`; one AI agent calls a slow external tool, checkpointing after each step so that killing and restarting the process mid-execution resumes from the last recorded checkpoint without re-running work that already completed.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `WeatherQueryAgent` (AutonomousAgent) performs the decision; the surrounding components prepare its input, gate its tool calls, and audit crash/resume events. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** (`ToolCallGuardrail`) validates every tool invocation before it executes — checking that the location argument is non-empty, does not contain injection-like tokens, and is within the allowed character set. A rejected call is returned as a structured error to the agent loop, which retries with a corrected argument.
- An **on-incident evaluator** (`IncidentEvaluator`) fires whenever the workflow detects a crash/resume transition. It records a structured `IncidentEvent` carrying the run id, the step that was interrupted, the elapsed time before crash, and a severity classification. The event is non-blocking: the workflow continues without waiting for the evaluator.

The blueprint shows that crash-resume durability is not a debugging curiosity — it is a first-class observable behaviour that deserves a governance hook.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a location (e.g., `"San Francisco, CA"`) into the **Location** field, picks a query type (current / forecast / air-quality), and clicks **Run agent**.
2. The UI POSTs to `/api/agent/run` and receives a `runId`.
3. The card appears in the live list in `INITIATED` state. Within ~1 s it transitions to `TOOL_CALLED` — the `SlowWeatherTool` is executing.
4. The status bar in the card shows an animated progress indicator during `TOOL_CALLED`. The developer is invited to kill the process now to test crash-resume.
5. After the tool call completes (or after restart), the workflow advances to `REPORTING`. Within ~5–30 s the card reaches `COMPLETED`. The `WeatherReport` appears in the right pane: a summary paragraph, temperature, conditions, and query timestamp.
6. If the run was resumed after a crash, a yellow **Resumed** badge appears on the card. The incident panel at the bottom of the right pane shows the `IncidentEvent` record.
7. If the `before-tool-call` guardrail blocked a call, a red **Guardrail** chip appears on the card with the rejection reason and the iteration count.
8. The user can start another run; the live list keeps prior runs visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AgentEndpoint` | `HttpEndpoint` | `/api/agent/*` — run, list, get, SSE; serves `/api/metadata/*`. | — | `AgentRunEntity`, `AgentRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AgentRunEntity` | `EventSourcedEntity` | Per-run lifecycle: initiated → tool_called → reporting → completed / failed / resumed. Source of truth. | `AgentEndpoint`, `AgentRunWorkflow` | `AgentRunView` |
| `AgentRunWorkflow` | `Workflow` | One workflow per run. Steps: `initStep` → `toolCallStep` → `reportStep`. Checkpointed; resumes from last completed step after crash. | started by `AgentEndpoint` on submit | `WeatherQueryAgent`, `AgentRunEntity` |
| `WeatherQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives a location + query type in the task definition; calls `SlowWeatherTool`; returns `WeatherReport`. | invoked by `AgentRunWorkflow` | returns report |
| `ToolCallGuardrail` | supporting class | `before-tool-call` guardrail registered on `WeatherQueryAgent`. Validates tool arguments before execution. | hooked on `WeatherQueryAgent` | rejects or passes through |
| `IncidentEvaluator` | supporting class | Fires after `RunResumed` event; scores the incident and emits `IncidentRecorded`. | `AgentRunWorkflow` via `evalStep` | `AgentRunEntity` |
| `AgentRunView` | `View` | Read model: one row per run for the UI. | `AgentRunEntity` events | `AgentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RunRequest(
    String runId,
    String location,
    QueryType queryType,
    String requestedBy,
    Instant requestedAt
) {}
enum QueryType { CURRENT, FORECAST, AIR_QUALITY }

record ToolCallAttempt(
    String toolName,
    String argumentLocation,
    QueryType queryType,
    Instant calledAt,
    boolean guardrailRejected,
    String rejectionReason   // null when not rejected
) {}

record WeatherReport(
    String location,
    QueryType queryType,
    String summary,
    double temperatureCelsius,
    String conditions,
    Instant reportedAt
) {}

record IncidentEvent(
    String runId,
    String interruptedStep,
    long elapsedBeforeCrashMs,
    IncidentSeverity severity,
    String notes,
    Instant detectedAt
) {}
enum IncidentSeverity { INFO, WARNING, ERROR }

record AgentRun(
    String runId,
    Optional<RunRequest> request,
    Optional<ToolCallAttempt> toolCall,
    Optional<WeatherReport> report,
    Optional<IncidentEvent> incident,
    AgentRunStatus status,
    int resumeCount,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AgentRunStatus {
    INITIATED, TOOL_CALLED, REPORTING, COMPLETED, FAILED, RESUMED
}
```

Events on `AgentRunEntity`: `RunInitiated`, `ToolCallStarted`, `ToolCallCompleted`, `ReportGenerated`, `RunCompleted`, `RunFailed`, `RunResumed`.

Every nullable lifecycle field on the `AgentRun` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/agent/run` — body `{ location, queryType, requestedBy }` → `{ runId }`.
- `GET /api/agent/runs` — list all runs, newest-first.
- `GET /api/agent/instances/{id}` — one run.
- `GET /api/agent/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: DurableAgentHTTP</title>`.

The App UI tab is a two-column layout: a left rail with the live list of agent runs (status pill + resume badge + guardrail chip + age) and a right pane with the selected run's detail — location, query type, tool-call status, weather report, and incident panel.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`ToolCallGuardrail`, registered on `WeatherQueryAgent` via the agent's guardrail-configuration block, bound to the `before-tool-call` hook): validates the `location` argument is non-empty, contains only printable characters, is ≤ 200 characters, and does not match injection-pattern heuristics. Validates `queryType` is one of `{CURRENT, FORECAST, AIR_QUALITY}`. On failure returns a structured rejection with the failed-check name; the agent loop retries within its iteration budget.
- **E1 — on-incident eval** (`IncidentEvaluator`, invoked as `incidentStep` inside `AgentRunWorkflow` when `RunResumed` is detected): records an `IncidentEvent` carrying `interruptedStep`, `elapsedBeforeCrashMs`, `severity` (INFO when the run had not yet called the tool; WARNING when the tool call was in-flight; ERROR when the report step was interrupted), and a notes string. Non-blocking — the workflow does not wait for this step to complete the `COMPLETED` transition.

## 9. Agent prompts

- `WeatherQueryAgent` → `prompts/weather-query-agent.md`. The single decision-making LLM. System prompt instructs it to call `SlowWeatherTool` with the supplied location and query type, then return a `WeatherReport`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User starts a run; within 30 s the report lands with a `WeatherReport` and an eval score visible on the card.
2. **J2** — Process is killed during `toolCallStep`; on restart the workflow resumes from `toolCallStep`, not from `initStep`; the `RunResumed` event and `IncidentEvent` appear; `resumeCount` increments.
3. **J3** — Agent submits a tool call with an empty location string; the `before-tool-call` guardrail rejects it; the agent retries with the correct location; the guardrail chip appears on the card.
4. **J4** — A run that was resumed has `IncidentSeverity.WARNING` in its incident panel because the tool call was in-flight at crash time.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named akka-resumable-agent-http demonstrating the single-agent × general
cell with crash-resume durability. Runs out of the box (no external services).
Maven group io.akka.samples.
Maven artifact single-agent-general-akka-resumable-agent-http.
Java package io.akka.samples.durableagentoverhttpwithcrashresume.
Akka 3.6.0.
HTTP port 9940.

Components to wire (exactly):

- 1 AutonomousAgent WeatherQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/weather-query-agent.md>) and
  .capability(TaskAcceptance.of(QUERY_WEATHER).maxIterationsPerTask(3)).
  The agent calls SlowWeatherTool (an in-process Tool that sleeps for a configurable
  delay — default 8 s — before returning a mock weather payload). The tool is
  registered on the agent via .tools(SlowWeatherTool.class). Output: WeatherReport.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the
  agent loop retries with a corrected argument within its 3-iteration budget.

- 1 Workflow AgentRunWorkflow per runId with four steps:
  * initStep — emits RunInitiated; advances to toolCallStep.
    WorkflowSettings.stepTimeout 5s.
  * toolCallStep — emits ToolCallStarted; then calls
    componentClient.forAutonomousAgent(WeatherQueryAgent.class, "agent-" + runId)
      .runSingleTask(TaskDef.instructions(formatQuery(request))) — returns a taskId;
    then forTask(taskId).result(QUERY_WEATHER) to fetch the WeatherReport; emits
    ToolCallCompleted. WorkflowSettings.stepTimeout 120s with
    defaultStepRecovery maxRetries(2).failoverTo(AgentRunWorkflow::error).
  * reportStep — emits ReportGenerated and RunCompleted; also checks whether
    AgentRunEntity.resumeCount > 0 and, if so, calls incidentStep first.
    WorkflowSettings.stepTimeout 30s.
  * incidentStep — invokes IncidentEvaluator.evaluate(runId, interruptedStep,
    elapsedBeforeCrashMs) → IncidentEvent; calls AgentRunEntity.recordIncident(event).
    WorkflowSettings.stepTimeout 10s.
  * error step — calls AgentRunEntity.fail(reason); WorkflowSettings.stepTimeout 5s.

  Crash-resume: Akka Workflow persists each step's completion. On process restart, the
  runtime replays from the last durable step boundary. This means toolCallStep is the
  key durability point: once toolCallStep persists ToolCallCompleted, a crash in
  reportStep does NOT re-invoke the agent or the tool.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AgentRunEntity (one per runId). State AgentRun{runId, request,
  toolCall, report, incident, status, resumeCount, createdAt, finishedAt}. AgentRunStatus
  enum: INITIATED, TOOL_CALLED, REPORTING, COMPLETED, FAILED, RESUMED. Events:
  RunInitiated{request}, ToolCallStarted{location, queryType}, ToolCallCompleted{toolResult},
  ReportGenerated{report}, RunCompleted{}, RunFailed{reason}, RunResumed{resumeCount}.
  Commands: initiate, startToolCall, completeToolCall, generateReport, complete, fail,
  resume, recordIncident, getRun.
  emptyState() returns AgentRun.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.
  resumeCount starts at 0; incremented by RunResumed event. Workflow calls resume()
  command on startup if the entity status is TOOL_CALLED or REPORTING (indicating
  the previous process died mid-step).

- 1 View AgentRunView with row type AgentRunRow (mirrors AgentRun minus raw request
  fields — keeps runId, location, queryType, status, resumeCount, report, incident,
  createdAt, finishedAt, guardrailRejected, rejectionReason). Table updater consumes
  AgentRunEntity events. ONE query getAllRuns: SELECT * AS runs FROM agent_run_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * AgentEndpoint at /api/agent with:
    POST /run (body {location, queryType, requestedBy}; mints runId; calls
      AgentRunEntity.initiate; starts AgentRunWorkflow; returns {runId}),
    GET /runs (list from getAllRuns, sorted newest-first),
    GET /instances/{id} (one row by runId),
    GET /runs/sse (Server-Sent Events from the view's stream-updates),
    three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AgentTasks.java declaring one Task<R> constant:
  QUERY_WEATHER = Task.name("Query weather").description("Call SlowWeatherTool with
  the supplied location and return a WeatherReport").resultConformsTo(WeatherReport.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- SlowWeatherTool.java — an in-process Tool implementation that sleeps for
  SlowWeatherTool.DELAY_SECONDS (configurable via application.conf; default 8) before
  returning a deterministic mock WeatherReport. The sleep is interruptible so the
  process-kill test (J2) can be demonstrated at any point during the delay.

- ToolCallGuardrail.java implementing the before-tool-call hook. Reads the candidate
  tool invocation (tool name + arguments), runs the four checks listed in eval-matrix.yaml
  G1 (non-empty location, ≤ 200 chars, printable characters only, no injection-pattern),
  and either passes through or returns Guardrail.reject(<structured-error-with-check-name>).

- IncidentEvaluator.java — pure deterministic logic (no LLM). Inputs: runId, interruptedStep
  name, elapsedBeforeCrashMs. Outputs: IncidentEvent with severity classification:
  INFO when interruptedStep is initStep (tool had not been called), WARNING when
  toolCallStep (tool call was in-flight), ERROR when reportStep. The scoring logic and
  severity table are documented in Javadoc on the class.

- Domain records RunRequest, ToolCallAttempt, WeatherReport, IncidentEvent, AgentRun,
  QueryType, AgentRunStatus, IncidentSeverity.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9940,
  slow-weather-tool.delay-seconds = 8 (overridable to 2 for fast tests), and the three
  model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-queries.jsonl with 4 seeded queries covering
  all three QueryType values and two different location formats (city name, city+state).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching Section 8.

- risk-survey.yaml at the project root.

- prompts/weather-query-agent.md loaded as the agent system prompt.

- README.md at the project root.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of run cards; right = selected-run detail with location, tool-call
  status, weather report, and incident panel).
  Browser title exactly: <title>Akka Sample: DurableAgentHTTP</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(runId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    query-weather.json — 6 WeatherReport entries covering all three QueryType values.
      Each entry has a summary paragraph, realistic temperatureCelsius value, and a
      conditions string. Plus 2 deliberately MALFORMED tool invocation attempts
      (one with empty location; one with invalid queryType value outside the enum)
      so the guardrail path (J3) is reproducible. The mock selects a malformed
      attempt on the FIRST iteration of every 4th run (modulo seed).
- MockModelProvider.seedFor(runId) makes per-run selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WeatherQueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AgentTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (toolCallStep 120s, initStep 5s, reportStep 30s, incidentStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on AgentRun is Optional<T>.
- Lesson 7: AgentTasks.java with QUERY_WEATHER mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9940 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- Single-agent invariant: exactly ONE AutonomousAgent (WeatherQueryAgent). IncidentEvaluator
  is rule-based (no LLM call).
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check. It fires before SlowWeatherTool executes.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
