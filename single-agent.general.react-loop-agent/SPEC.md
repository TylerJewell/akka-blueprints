# SPEC — react-loop-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ReAct Agent Workflow.
**One-line pitch:** A user submits a query and a list of available tools; one AI agent interleaves Thought/Action/Observation steps — calling tools when needed — until it produces a final answer or exhausts its iteration budget.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `ReActAgent` (AutonomousAgent) owns the full reasoning loop; the surrounding components register tools, persist every step, and audit the finished chain. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before every tool invocation — verifying the tool exists in the registered set, that the call parameters conform to the tool's schema, and that the tool is not on the run's deny list. A blocked call returns a structured error observation that the agent receives and must reason past; the run continues.
- An **on-decision eval** runs immediately after the final answer is recorded, scoring the reasoning chain for hallucinated steps (a Thought that contradicts a prior Observation), circular loops (the same thought/tool pair repeated without progress), and unsupported claims (a claim in the final answer that no Observation supports).

The blueprint shows that even a free-running reasoning loop can carry independent governance: the tool-call guardrail sits at the action boundary, and the chain evaluator audits the completed trace without stopping the run.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **query** in the text area (or picks one of three seeded examples — a multi-step arithmetic problem, a research synthesis task, and a conditional lookup task).
2. The user selects which **tools** to make available from a checklist (Calculator, WebLookup, DataFetch — all in-process stubs).
3. The user clicks **Run**. The UI POSTs to `/api/runs` and receives a `runId`.
4. The run card appears in the live list in `PENDING` state. Within ~1 s it transitions to `RUNNING` as the workflow starts. The step trace begins to fill in real-time via SSE: each Thought, Action, and Observation appears as it is committed to the entity.
5. When the agent returns a final answer, the card transitions to `COMPLETED`. The right pane shows the final answer text and the full numbered step trace.
6. Within ~1 s of `COMPLETED`, the eval finishes. The card shows a **chain score** chip (1–5) and a one-line rationale describing the quality of the reasoning chain.
7. The user can submit another query; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `RunEntity`, `RunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RunEntity` | `EventSourcedEntity` | Per-run lifecycle: pending → running → completed / exhausted / failed. Full step trace. Source of truth. | `RunEndpoint`, `ToolDispatcher`, `ReActWorkflow` | `RunView` |
| `ToolDispatcher` | `Consumer` | Subscribes to `ActionRequested` events; dispatches to the named tool stub; emits `ObservationRecorded` back to the entity. | `RunEntity` events | `RunEntity` |
| `ReActWorkflow` | `Workflow` | One workflow per run. Steps: `initStep` → `loopStep` (repeating) → `finalizeStep`. | started by `RunEndpoint` once run is accepted | `ReActAgent`, `RunEntity` |
| `ReActAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the query and available-tool manifest in the task definition; emits Steps (Thought/Action/Answer) one at a time via the task attachment interface. | invoked by `ReActWorkflow` | returns `ReActResult` |
| `ToolCallGuardrail` | supporting class | Validates every tool call before dispatch: tool registered, params schema-valid, tool not denied. Returns a structured block observation on failure. | invoked by `ReActWorkflow.loopStep` before calling `ToolDispatcher` | returns allow / block |
| `ChainEvaluator` | supporting class | Deterministic scorer on the finished step trace. No LLM call. Checks for circular loops, unsupported claims, and hallucinated steps. | invoked by `ReActWorkflow.finalizeStep` | returns `EvalResult` |
| `RunView` | `View` | Read model: one row per run for the UI. | `RunEntity` events | `RunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolSpec(String toolName, String description, String paramSchema) {}

record RunRequest(
    String runId,
    String query,
    List<ToolSpec> availableTools,
    List<String> deniedTools,
    String submittedBy,
    Instant submittedAt
) {}

record Step(
    int stepIndex,
    StepKind kind,     // THOUGHT, ACTION, OBSERVATION, ANSWER
    String content,    // thought text / action call / observation / final answer
    String toolName,   // non-null for ACTION and OBSERVATION
    boolean blocked,   // true when ToolCallGuardrail blocked the action
    Instant recordedAt
) {}
enum StepKind { THOUGHT, ACTION, OBSERVATION, ANSWER }

record ReActResult(
    String finalAnswer,
    List<Step> steps,
    Outcome outcome,
    Instant decidedAt
) {}
enum Outcome { ANSWERED, EXHAUSTED }

record EvalResult(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Run(
    String runId,
    Optional<RunRequest> request,
    List<Step> steps,
    Optional<ReActResult> result,
    Optional<EvalResult> eval,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus {
    PENDING, RUNNING, COMPLETED, EXHAUSTED, FAILED
}
```

Events on `RunEntity`: `RunSubmitted`, `RunStarted`, `StepRecorded`, `ActionBlocked`, `RunCompleted`, `RunExhausted`, `EvaluationScored`, `RunFailed`.

Every nullable lifecycle field on the `Run` state record is `Optional<T>` (Lesson 6). `steps` is never null — it starts as an empty list and grows.

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ query, availableTools: [ToolSpec], deniedTools: [String], submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per step recorded.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ReAct Agent Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted runs (status pill + outcome badge + age) and a right pane with the selected run's detail — the query, the available tools checklist, the live step trace (Thought/Action/Observation/Answer chips), the final answer block, and the chain-eval score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`ToolCallGuardrail`, invoked by `ReActWorkflow.loopStep` before handing the action to `ToolDispatcher`): checks (1) the tool name exists in `RunRequest.availableTools`, (2) the call parameters are valid JSON conforming to the tool's `paramSchema`, and (3) the tool name is not in `RunRequest.deniedTools`. On failure, returns a structured block observation to the agent so it can adjust its action plan. The agent's iteration count still advances; a blocked action consumes one step toward the budget. The original action call is recorded as an `ActionBlocked` event on the entity so the audit trail is complete.
- **E1 — on-decision eval** (`ChainEvaluator`, `eval-event`, `on-decision-eval`): runs in `finalizeStep` immediately after `RunCompleted` or `RunExhausted` lands. Deterministic, no LLM call. Checks: (a) no Thought text is a verbatim repeat of an earlier Thought (circular loop), (b) every tool-name claim in the final answer is supported by at least one `OBSERVATION` step with that tool, (c) the step-count ratio (ACTION steps / total steps) is reasonable (> 0.1 — a chain that is all Thoughts and no Actions is suspect). Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `ReActAgent` → `prompts/react-agent.md`. The single decision-making LLM. System prompt instructs it to follow the Thought/Action/Observation format strictly, pick tools by exact `toolName`, and emit an ANSWER step when it has enough information.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the arithmetic seed query with Calculator enabled; within 30 s the final answer appears with a numbered step trace showing at least two Action/Observation pairs and an eval score chip.
2. **J2** — User submits a query with `DataFetch` in the deny list; the agent attempts to call `DataFetch`; the guardrail returns a block observation; the agent reasons past it and completes the query using permitted tools only. The `ActionBlocked` event is visible in the step trace.
3. **J3** — A run whose step trace contains a repeated Thought receives an eval score of 1–2 with rationale citing the circular step; the UI flags the card.
4. **J4** — A run that reaches the iteration budget without returning an ANSWER step transitions to `EXHAUSTED`; the UI shows the partial trace with an `EXHAUSTED` status pill.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named react-loop-agent demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-react-loop-agent. Java package io.akka.samples.reactagentworkflow.
Akka 3.6.0. HTTP port 9702.

Components to wire (exactly):

- 1 AutonomousAgent ReActAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/react-agent.md>) and
  .capability(TaskAcceptance.of(RUN_QUERY).maxIterationsPerTask(10)). The task
  receives the query and available-tool manifest in the task instructions; the
  agent emits one Step per iteration (Thought, then Action, then waits for
  an Observation before the next Thought). Output: ReActResult{finalAnswer: String,
  steps: List<Step>, outcome: Outcome (ANSWERED/EXHAUSTED), decidedAt: Instant}.
  No before-agent-response guardrail on the agent itself — governance sits at the
  tool-call boundary (G1) and on the finished chain (E1).

- 1 Workflow ReActWorkflow per runId with four steps:
  * initStep — calls RunEntity.markRunning; emits RunStarted. Advances immediately
    to loopStep. WorkflowSettings.stepTimeout 5s.
  * loopStep — calls componentClient.forAutonomousAgent(ReActAgent.class,
    "agent-" + runId).runSingleTask(TaskDef.instructions(formatQuery(run.request)))
    to obtain the next Step from the agent. Before forwarding any ACTION step to
    ToolDispatcher, calls ToolCallGuardrail.check(step, run.request) — on block,
    records ActionBlocked and emits a synthetic OBSERVATION step back to the agent.
    On THOUGHT or OBSERVATION steps, records StepRecorded. On ANSWER step, advances
    to finalizeStep. If stepIndex >= 10 with no ANSWER, advances to finalizeStep
    with outcome EXHAUSTED. WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(ReActWorkflow::error).
  * finalizeStep — calls RunEntity.complete(result). Then runs ChainEvaluator
    .score(run.steps, run.result) and calls RunEntity.recordEvaluation(eval).
    WorkflowSettings.stepTimeout 10s.
  * error step — calls RunEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a
  top-level WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity RunEntity (one per runId). State Run{runId: String,
  request: Optional<RunRequest>, steps: List<Step>, result: Optional<ReActResult>,
  eval: Optional<EvalResult>, status: RunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RunStatus enum: PENDING, RUNNING, COMPLETED,
  EXHAUSTED, FAILED. Events: RunSubmitted{request}, RunStarted{}, StepRecorded{step},
  ActionBlocked{step, reason}, RunCompleted{result}, RunExhausted{result},
  EvaluationScored{eval}, RunFailed{reason}. Commands: submit, markRunning,
  recordStep, blockAction, complete, exhaust, recordEvaluation, fail, getRun.
  emptyState() returns Run.initial("") with steps = List.of(), all Optional fields
  = Optional.empty(), no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the
  event-applier. steps list is immutable-accumulated (replaceState with
  appended list).

- 1 Consumer ToolDispatcher subscribed to RunEntity events; on ActionRequested
  (i.e. a StepRecorded event where step.kind == ACTION and !step.blocked), parses
  step.content as { toolName, params }, dispatches to the matching in-process stub
  (CalculatorTool, WebLookupTool, DataFetchTool), captures the observation text,
  and calls RunEntity.recordStep(observationStep). Tool stubs live in
  application/tools/. Each stub has a name(), paramSchema(), and execute(params)
  method returning a String observation. Unknown tool names return an error
  observation (do not throw).

- 1 View RunView with row type RunRow (mirrors Run; steps list is included in full
  so the UI can render the trace without a separate fetch). Table updater consumes
  RunEntity events. ONE query getAllRuns: SELECT * AS runs FROM run_view. No WHERE
  filter — caller filters client-side (Lesson 2).

- 2 HttpEndpoints:
  * RunEndpoint at /api with POST /runs (body {query, availableTools: [{toolName,
    description, paramSchema}], deniedTools: [String], submittedBy}; mints runId;
    calls RunEntity.submit; starts ReActWorkflow; returns {runId}), GET /runs
    (list from getAllRuns, newest-first), GET /runs/{id} (one row), GET /runs/sse
    (Server-Sent Events forwarded from view's stream-updates), and three
    /api/metadata/* endpoints.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- RunTasks.java declaring one Task<R> constant: RUN_QUERY = Task.name("Run query")
  .description("Execute the ReAct loop for the given query using the available tools")
  .resultConformsTo(ReActResult.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records ToolSpec, RunRequest, Step, StepKind, ReActResult, Outcome,
  EvalResult, Run, RunStatus.

- ToolCallGuardrail.java — a plain Java class (NOT a guardrail hook on the agent
  itself). Invoked synchronously by ReActWorkflow.loopStep before forwarding an
  ACTION step. Checks: (1) toolName present in run.request.availableTools, (2)
  params is valid JSON, (3) toolName not in run.request.deniedTools. Returns a
  GuardrailOutcome{allowed: boolean, blockReason: String}. On block, the workflow
  records ActionBlocked and synthesizes an OBSERVATION step with content =
  "[BLOCKED] " + blockReason.

- ChainEvaluator.java — pure deterministic logic (no LLM). Inputs: List<Step> and
  ReActResult. Outputs: EvalResult. Scoring rubric: starts at 5; deduct 2 if any
  THOUGHT text is a verbatim repeat of a prior THOUGHT; deduct 1 if the final answer
  claims a tool was used but no matching OBSERVATION step exists; deduct 1 if
  ACTION step count divided by total step count is less than 0.1 (thought-only
  chain). Clamp to 1 minimum. Rubric documented in Javadoc on the class.

- application/tools/CalculatorTool.java, WebLookupTool.java, DataFetchTool.java —
  in-process stubs. Calculator evaluates simple arithmetic expressions (no eval();
  use a safe parser). WebLookup returns a canned paragraph keyed by a keyword in
  the query. DataFetch returns a canned JSON snippet keyed by a record id.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9702
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-queries.jsonl with 3 seeded queries:
  a 3-step arithmetic problem requiring Calculator twice, a research question
  requiring WebLookup then DataFetch, and a conditional lookup requiring DataFetch
  then Calculator.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching Section 8.

- risk-survey.yaml at project root per Section 8 defaults.

- prompts/react-agent.md loaded as the agent system prompt.

- README.md at project root per Section 1.

- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs. App UI tab uses a two-column layout (left = run
  submission panel + live run list; right = selected run detail with query,
  tool checklist, live step trace, final answer block, and chain-eval score chip).
  Browser title exactly: <title>Akka Sample: ReAct Agent Workflow</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct ReActResult outputs per run (seeded by runId).
        Mock entries include: runs that ANSWER after 3 steps, runs that EXHAUST
        at step 10, runs with circular Thoughts, and runs with blocked tool calls.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.

Constraints:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ReActAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. RunTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (initStep 5s, loopStep
  60s, finalizeStep 10s, error 5s).
- Lesson 6: every Optional<T> field on Run uses Optional.empty() initially and
  Optional.of(...) in event-appliers. steps list starts as List.of().
- Lesson 7: RunTasks.java with RUN_QUERY = Task.name(...).description(...)
  .resultConformsTo(ReActResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9702 declared in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and
  themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute, never by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (ReActAgent). The
  ToolCallGuardrail and ChainEvaluator are deterministic Java classes — no LLM call.
- ToolCallGuardrail is NOT a before-agent-response hook on the agent. It is called
  explicitly by loopStep before dispatching an ACTION step. This keeps the agent's
  own loop clean and places governance at the tool boundary.
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
