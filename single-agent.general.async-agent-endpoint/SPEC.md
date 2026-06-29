# SPEC — async-agent-endpoint

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** AsyncAgentEndpoint.
**One-line pitch:** A user submits a natural-language task; one AI agent generates Python code to satisfy it and returns a typed `RunResult` — the Starlette-style endpoint dispatches each agent run off the event loop so concurrent requests stay responsive.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `CodeRunnerAgent` (AutonomousAgent) carries the entire decision; the surrounding components only queue its input and persist its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** intercepts every tool invocation the agent emits before execution. It inspects the generated Python code for forbidden imports (`os.system`, `subprocess`, `socket`, and raw `__import__`) and rejects any call that violates the sandbox policy. A rejected tool call triggers a retry inside the same task, giving the agent a chance to rewrite the code without the forbidden construct.

The blueprint shows that even a general-purpose code-running agent can be governed without a heavyweight external sandbox — the guardrail acts as a fast, in-process policy check that lets safe tool calls through immediately while blocking only the unsafe ones.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types (or picks from three seeded examples) a natural-language **task prompt** describing what the code should compute: a string transformation, a data-parsing task, or a math computation.
2. The user clicks **Run task**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `RUNNING` as the workflow dispatches the agent via `anyio.to_thread.run_sync`.
4. Within ~10–30 s, the `runStep` completes. The card transitions to `COMPLETED`. The result appears: the generated code in a monospace block, the captured stdout/stderr, and the final output value.
5. If the agent's generated code triggered the guardrail, the card shows a `GUARDRAIL_HIT` badge for that iteration (the retry succeeded; the badge is informational).
6. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AgentRunEntity`, `RunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AgentRunEntity` | `EventSourcedEntity` | Per-run lifecycle: submitted → running → completed / failed. Source of truth. | `RunEndpoint`, `AgentRunWorkflow` | `RunView` |
| `AgentRunWorkflow` | `Workflow` | One workflow per run. Steps: `runStep` → (on success) `recordStep`; error step transitions entity to FAILED. | started by `RunEndpoint` immediately after entity submit | `CodeRunnerAgent`, `AgentRunEntity` |
| `CodeRunnerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task prompt and any context; generates Python code; executes it via its tool; returns `RunResult`. | invoked by `AgentRunWorkflow` | returns result |
| `SandboxGuardrail` | — (guardrail hook class) | Intercepts every tool call the agent emits; inspects generated code for forbidden constructs; rejects violations. | wired into `CodeRunnerAgent` | allows or rejects |
| `RunView` | `View` | Read model: one row per run for the UI. | `AgentRunEntity` events | `RunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TaskPrompt(
    String taskId,
    String promptText,
    String submittedBy,
    Instant submittedAt
) {}

record GeneratedCode(
    String code,
    String language    // always "python" in this blueprint
) {}

enum RunStatus { SUBMITTED, RUNNING, COMPLETED, FAILED }

enum FailureKind { GUARDRAIL_EXHAUSTED, EXECUTION_ERROR, TIMEOUT }

record RunResult(
    String stdout,
    String stderr,
    String outputValue,       // final expression value or empty string
    boolean guardrailHitOnAnyIteration,
    GeneratedCode code,
    Instant completedAt
) {}

record RunFailure(
    FailureKind kind,
    String message,
    Instant failedAt
) {}

record AgentRun(
    String runId,
    Optional<TaskPrompt> prompt,
    Optional<RunResult> result,
    Optional<RunFailure> failure,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}
```

Events on `AgentRunEntity`: `RunSubmitted`, `RunStarted`, `RunCompleted`, `RunFailed`.

Every nullable lifecycle field on the `AgentRun` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ promptText, submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: AsyncAgentEndpoint</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted runs (status pill + guardrail-hit badge + age) and a right pane with the selected run's detail — submitted prompt, generated code block, stdout/stderr, output value, and any guardrail-hit annotation.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call emitted by `CodeRunnerAgent`. Before the tool executes, `SandboxGuardrail` inspects the `code` argument for forbidden constructs: any use of `os.system`, `subprocess` (all forms), `socket`, raw `__import__`, and `eval` / `exec` on externally-sourced strings. On a violation, returns a structured `policy-violation` rejection with the name of the forbidden construct; the agent loop consumes one of its iterations and retries. Permitted calls flow through immediately. The guardrail is registered on the agent's guardrail-configuration block, bound to the `before-tool-call` hook.

## 9. Agent prompts

- `CodeRunnerAgent` → `prompts/code-runner.md`. The single decision-making LLM. System prompt instructs it to generate Python code to satisfy the user's task, call its `execute_code` tool with the generated code, and return a `RunResult` capturing the output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seeded string-transformation task; within 30 s the result appears with generated code, stdout, and output value.
2. **J2** — The agent generates code importing `subprocess` on its first iteration (mock LLM path) — the `before-tool-call` guardrail rejects it; the second iteration generates compliant code; the UI shows the completed result with a `GUARDRAIL_HIT` badge.
3. **J3** — A task prompt that causes a runtime exception in the generated code is recorded as `FAILED` with `FailureKind.EXECUTION_ERROR` and the exception message.
4. **J4** — Two runs submitted back-to-back both complete without the second waiting for the first — event loop non-blocking confirmed by overlapping timestamps.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named async-agent-endpoint demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-async-agent-endpoint. Java package io.akka.samples.asyncagentstarlette.
Akka 3.6.0. HTTP port 9526.

Components to wire (exactly):

- 1 AutonomousAgent CodeRunnerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/code-runner.md>) and
  .capability(TaskAcceptance.of(RUN_TASK).maxIterationsPerTask(3)). The task receives the
  natural-language prompt as its instruction text. Output: RunResult{stdout: String,
  stderr: String, outputValue: String, guardrailHitOnAnyIteration: boolean,
  code: GeneratedCode, completedAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  3-iteration budget.

- 1 Workflow AgentRunWorkflow per runId with two steps:
  * runStep — emits RunStarted, then calls componentClient.forAutonomousAgent(
    CodeRunnerAgent.class, "runner-" + runId).runSingleTask(
      TaskDef.instructions(prompt.promptText())
    ) — returns a taskId, then forTask(taskId).result(RUN_TASK) to fetch the RunResult.
    On success calls AgentRunEntity.recordCompleted(result). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(1).failoverTo(AgentRunWorkflow::error).
  * error step — calls AgentRunEntity.recordFailed(reason) and transitions entity to
    FAILED. WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AgentRunEntity (one per runId). State AgentRun{runId: String,
  prompt: Optional<TaskPrompt>, result: Optional<RunResult>, failure: Optional<RunFailure>,
  status: RunStatus, createdAt: Instant, finishedAt: Optional<Instant>}. RunStatus enum:
  SUBMITTED, RUNNING, COMPLETED, FAILED. Events: RunSubmitted{prompt}, RunStarted{},
  RunCompleted{result}, RunFailed{failure}. Commands: submit, markRunning, recordCompleted,
  recordFailed, getRun. emptyState() returns AgentRun.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 View RunView with row type RunRow (mirrors AgentRun fields). Table updater consumes
  AgentRunEntity events. ONE query getAllRuns: SELECT * AS runs FROM run_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * RunEndpoint at /api with POST /runs (body {promptText, submittedBy}; mints runId;
    calls AgentRunEntity.submit; starts AgentRunWorkflow with id = "run-" + runId;
    returns {runId}), GET /runs (list from getAllRuns, sorted newest-first), GET /runs/{id}
    (one row), GET /runs/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- RunTasks.java declaring one Task<R> constant: RUN_TASK = Task.name("Run task").description(
  "Generate Python code to satisfy the task prompt, execute it, and return a RunResult")
  .resultConformsTo(RunResult.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records TaskPrompt, GeneratedCode, RunResult, RunFailure, AgentRun, RunStatus,
  FailureKind.

- SandboxGuardrail.java implementing the before-tool-call hook. Reads the tool call's
  code argument, runs a pattern scan for os.system, subprocess (module import or call),
  socket (module import or call), __import__ (raw call), and eval/exec on non-literal
  strings. On any match returns Guardrail.reject(<structured policy-violation>) naming
  the forbidden construct. Compliant calls pass through.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9526 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  CodeRunnerAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded task prompts:
  (1) "Count the number of vowels in the string 'supercalifragilistic'",
  (2) "Parse the CSV 'name,age\nAlice,30\nBob,25' and return the average age as a float",
  (3) "Compute the first 10 Fibonacci numbers and return them as a comma-separated string".

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root pre-filled for the general domain as described
  in Section 5 of this SPEC; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/code-runner.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: AsyncAgentEndpoint", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of run cards; right = selected-run detail with submitted prompt,
  generated code block, stdout/stderr, output value, guardrail-hit annotation).
  Browser title exactly: <title>Akka Sample: AsyncAgentEndpoint</title>. No subtitle on
  the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(runId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    run-task.json — 6 RunResult entries covering COMPLETED runs with varied outputs
      (string result, numeric result, multi-line stdout). Each entry has non-empty
      generatedCode.code, non-empty stdout, and a non-empty outputValue. Plus 2
      deliberately POLICY-VIOLATING tool call entries (one importing subprocess, one
      calling os.system) — the guardrail blocks both, exercising the retry path. The
      mock should select a violating entry on the FIRST iteration of every 3rd run
      (modulo seed) so J2 is reproducible.
    Plus 1 EXECUTION_ERROR entry that returns a RunResult with a non-empty stderr and an
      empty outputValue, allowing J3 to be exercised by selecting a specific runId seed.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeRunnerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RunTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (runStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the AgentRun record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: RunTasks.java with RUN_TASK = Task.name(...).description(...)
  .resultConformsTo(RunResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9526 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeRunnerAgent).
  The sandbox guardrail is a policy-check class (SandboxGuardrail), not a second agent.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the tool returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference for how the hook is registered.
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
