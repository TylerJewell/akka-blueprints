# SPEC — sandboxed-code-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SandboxedCodeAgent.
**One-line pitch:** A user submits a natural-language task; one AI agent generates Python to solve it and executes that code inside an isolated sandbox — Playwright, E2B, Modal, or Docker — subject to a blocking guardrail on every tool call and an automatic safety halt for runaway processes.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `CodeExecutionAgent` (AutonomousAgent) carries the entire task: understand the request, generate code, call the `execute_code` tool, and return a typed `ExecutionResult`. The surrounding components route the request, screen the generated code, manage the sandbox back-end, and record outcomes. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs on every `execute_code` tool invocation. It screens the generated Python for forbidden patterns (host-path traversal, network exfiltration, environment variable harvesting, arbitrary subprocess spawning) and blocks the call if any pattern matches. A blocked call returns a structured rejection that the agent loop sees as a tool error; the agent must revise the code and retry.
- An **automatic safety halt** terminates sandbox processes that exceed their resource budget (wall-clock timeout or CPU-second limit). The `SafetyHaltMonitor` Consumer polls the sandbox back-end's resource channel; on breach it emits `ExecutionHalted`, which the workflow detects and uses to transition the entity to `HALTED`.

The blueprint shows that the single-agent code-execution pattern can carry enforceable code-safety guarantees — the guardrail and halt mechanism together form a two-layer kill switch around every generated program.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types or selects a task in the **Task** textarea (or picks one of three seeded examples — a data-analysis task, a file-transformation task, a web-screenshot task).
2. The user picks a **Sandbox back-end** from a dropdown (Docker / E2B / Modal / Playwright) and optionally sets a **Wall-clock budget** (default 30 s) and **CPU-second budget** (default 10 s).
3. The user clicks **Run task**. The UI POSTs to `/api/executions` and receives an `executionId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SCREENING` — the guardrail is evaluating the generated code.
5. On approval, the card transitions to `RUNNING`. Within the configured budget, it transitions to `COMPLETED`. The result panel shows the stdout output, any generated files (listed by name), the exit code, and the wall-clock duration.
6. If the guardrail blocks the first attempt, the card stays in `SCREENING` and shows a `BLOCKED` chip with the violated-pattern name. On the agent's approved retry, the chip clears and `RUNNING` appears.
7. If the sandbox exceeds its budget, the card transitions to `HALTED` and shows the resource limit that was breached.
8. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ExecutionEndpoint` | `HttpEndpoint` | `/api/executions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ExecutionEntity`, `ExecutionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ExecutionEntity` | `EventSourcedEntity` | Per-execution lifecycle: submitted → screening → running → completed/halted/failed. Source of truth. | `ExecutionEndpoint`, `SandboxRouter`, `ExecutionWorkflow` | `ExecutionView` |
| `SandboxRouter` | `Consumer` | Subscribes to `CodeApproved` events; dispatches to the configured sandbox back-end (Docker/E2B/Modal/Playwright); calls `ExecutionEntity.recordOutcome` on completion. | `ExecutionEntity` events | `ExecutionEntity` |
| `ExecutionWorkflow` | `Workflow` | One workflow per execution. Steps: `screenStep` → `runStep` → `recordStep`. | started by `ExecutionEndpoint` after submit | `CodeExecutionAgent`, `ExecutionEntity`, `SafetyHaltMonitor` |
| `CodeExecutionAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task description plus sandbox constraints; generates Python; calls the `execute_code` tool; returns `ExecutionResult`. | invoked by `ExecutionWorkflow` | returns result |
| `CodeScreeningGuardrail` | supporting class | Registered as `before-tool-call` hook on `CodeExecutionAgent`. Screens generated Python for forbidden patterns; blocks or passes each tool call. | invoked inside agent loop | agent loop |
| `SafetyHaltMonitor` | `Consumer` | Polls sandbox resource usage; on budget breach emits `ExecutionHalted` to `ExecutionEntity`. | sandbox back-end resource channel | `ExecutionEntity` |
| `ExecutionView` | `View` | Read model: one row per execution for the UI. | `ExecutionEntity` events | `ExecutionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ExecutionRequest(
    String executionId,
    String taskDescription,
    SandboxBackend backend,
    int wallClockBudgetSeconds,
    int cpuBudgetSeconds,
    String submittedBy,
    Instant submittedAt
) {}
enum SandboxBackend { DOCKER, E2B, MODAL, PLAYWRIGHT }

record GeneratedCode(
    String pythonSource,
    String codeHash    // SHA-256 hex, minted by the agent after generation
) {}

record ScreeningOutcome(
    ScreeningDecision decision,
    String violatedPattern,  // null when APPROVED
    String rationale
) {}
enum ScreeningDecision { APPROVED, BLOCKED }

record SandboxOutput(
    int exitCode,
    String stdout,
    String stderr,
    List<String> generatedFiles,
    long wallClockMs,
    long cpuMs
) {}

record HaltReason(
    String limitBreached,    // "wall-clock" | "cpu"
    long measuredMs,
    long budgetMs
) {}

record ExecutionResult(
    ExecutionOutcome outcome,
    Optional<SandboxOutput> output,
    Optional<HaltReason> haltReason,
    Optional<ScreeningOutcome> lastScreening,
    Instant finishedAt
) {}
enum ExecutionOutcome { COMPLETED, HALTED, FAILED }

record Execution(
    String executionId,
    Optional<ExecutionRequest> request,
    Optional<GeneratedCode> code,
    Optional<ScreeningOutcome> screening,
    Optional<SandboxOutput> output,
    Optional<HaltReason> haltReason,
    ExecutionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExecutionStatus {
    SUBMITTED, SCREENING, APPROVED, RUNNING, COMPLETED, HALTED, FAILED
}
```

Events on `ExecutionEntity`: `CodeSubmitted`, `CodeGenerated`, `CodeApproved`, `CodeBlocked`, `ExecutionStarted`, `ExecutionCompleted`, `ExecutionHalted`, `ExecutionFailed`.

Every nullable lifecycle field on the `Execution` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/executions` — body `{ taskDescription, backend, wallClockBudgetSeconds, cpuBudgetSeconds, submittedBy }` → `{ executionId }`.
- `GET /api/executions` — list all executions, newest-first.
- `GET /api/executions/{id}` — one execution.
- `GET /api/executions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Sandboxed Code Execution</title>`.

The App UI tab is a two-column layout: a left rail with the submission form and the live list of executions (status pill + outcome badge + age) and a right pane with the selected execution's detail — task description, sandbox config, generated code, screening outcome, stdout/stderr output, generated files, and halt reason (when applicable).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `execute_code` tool call made by `CodeExecutionAgent`. `CodeScreeningGuardrail` is registered via the agent's guardrail-configuration block, bound to the `before-tool-call` hook. On each candidate tool invocation it (1) extracts the `code` parameter, (2) runs a set of pattern matchers covering host-path traversal (`/etc/`, `/proc/`, `~/.ssh/`), network exfiltration (outbound socket calls, `requests.get` to non-localhost), environment variable harvesting (`os.environ`, `subprocess` reading env), and dynamic code execution (`exec(`, `eval(`, `__import__`), and (3) either passes the call through or returns `Guardrail.reject(<pattern-name>)` so the agent loop receives a structured tool error and must revise. On guardrail rejection the agent retries within its 4-iteration budget.
- **H1 — automatic safety halt**: `SafetyHaltMonitor` Consumer subscribes to `ExecutionStarted` events. For each running execution it polls the sandbox back-end's process-stats channel every 2 s. When `wallClockMs > wallClockBudgetSeconds * 1000` or `cpuMs > cpuBudgetSeconds * 1000`, it calls the sandbox back-end's `kill` method and calls `ExecutionEntity.halt(haltReason)`. The entity emits `ExecutionHalted` and transitions to `HALTED`. This mechanism is independent of the agent loop — it fires based on observed resource usage, not on agent behaviour.

## 9. Agent prompts

- `CodeExecutionAgent` → `prompts/code-execution-agent.md`. The single decision-making LLM. System prompt instructs it to understand the task, generate self-contained Python, submit it via the `execute_code` tool, and interpret the result. The prompt explicitly forbids accessing the host filesystem, exfiltrating data, or reading environment variables.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the data-analysis seed task against the Docker back-end; within 30 s the result appears with stdout output, exit code 0, and wall-clock duration.
2. **J2** — The agent generates code containing `os.environ.get("ANTHROPIC_API_KEY")`; the `before-tool-call` guardrail blocks it with pattern `env-harvest`; the agent revises to remove the env read; the second submission is approved and runs to completion. The UI shows a `BLOCKED` chip for the first attempt and clears it on the approved retry.
3. **J3** — A task whose Python enters an infinite loop exceeds the wall-clock budget; `SafetyHaltMonitor` detects the breach and kills the sandbox process; the entity records `ExecutionHalted`; the UI shows a `HALTED` card with `limit_breached: "wall-clock"`.
4. **J4** — The LLM call log does not contain any value of `E2B_API_KEY` or `ANTHROPIC_API_KEY` from the host environment — only the task description and generated code appear in the agent's context.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named sandboxed-code-agent demonstrating the single-agent × dev-code cell.
Runs out of the box with Docker (no external services required for the Docker back-end).
Maven group io.akka.samples. Maven artifact single-agent-dev-code-sandboxed-code-agent.
Java package io.akka.samples.sandboxedcodeexecution. Akka 3.6.0. HTTP port 9103.

Components to wire (exactly):

- 1 AutonomousAgent CodeExecutionAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/code-execution-agent.md>) and
  .capability(TaskAcceptance.of(EXECUTE_TASK).maxIterationsPerTask(4)). The agent exposes
  one tool: execute_code(code: String, language: String = "python") which is a stub that
  forwards to the SandboxDispatcher helper. Before each execute_code invocation,
  CodeScreeningGuardrail runs as a before-tool-call hook registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow ExecutionWorkflow per executionId with three steps:
  * screenStep — invokes CodeExecutionAgent via componentClient.forAutonomousAgent(
    CodeExecutionAgent.class, "agent-" + executionId).runSingleTask(
      TaskDef.instructions(request.taskDescription())
    ). WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ExecutionWorkflow::error).
  * runStep — reads ExecutionEntity to confirm CodeApproved status, then calls
    SandboxDispatcher.dispatch(executionId, code, backend, budgets). Emits ExecutionStarted.
    WorkflowSettings.stepTimeout equal to (wallClockBudgetSeconds + 10) seconds minimum 15s.
  * recordStep — calls ExecutionEntity.recordOutcome(result). WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ExecutionEntity (one per executionId). State Execution{executionId:
  String, request: Optional<ExecutionRequest>, code: Optional<GeneratedCode>, screening:
  Optional<ScreeningOutcome>, output: Optional<SandboxOutput>, haltReason: Optional<HaltReason>,
  status: ExecutionStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  ExecutionStatus enum: SUBMITTED, SCREENING, APPROVED, RUNNING, COMPLETED, HALTED, FAILED.
  Events: CodeSubmitted{request}, CodeGenerated{code}, CodeApproved{screening},
  CodeBlocked{screening}, ExecutionStarted{}, ExecutionCompleted{output},
  ExecutionHalted{haltReason}, ExecutionFailed{reason}. Commands: submit, recordCodeGenerated,
  approveCode, blockCode, markRunning, recordOutcome, halt, fail, getExecution.
  emptyState() returns Execution.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer SafetyHaltMonitor subscribed to ExecutionEntity events; on ExecutionStarted
  starts polling the sandbox back-end's resource channel every 2000ms. When wallClockMs
  exceeds budget or cpuMs exceeds budget, calls the sandbox's kill() method and then
  calls ExecutionEntity.halt(haltReason). Stops polling on ExecutionCompleted,
  ExecutionHalted, or ExecutionFailed. Polling is implemented with a scheduled virtual-thread
  task; do not use Akka Streams or ActorSystem for this.

- 1 Consumer SandboxRouter subscribed to CodeApproved events. On CodeApproved, reads the
  configured SANDBOX_BACKEND env var (default: DOCKER). Routes to the matching
  SandboxDispatcher implementation: DockerSandboxDispatcher, E2BSandboxDispatcher,
  ModalSandboxDispatcher, PlaywrightSandboxDispatcher. Each dispatcher implements
  interface SandboxDispatcher { SandboxOutput run(String code, SandboxConfig config); }.
  DockerSandboxDispatcher uses docker run with a python:3.12-slim image, tmpfs mount,
  --network none, --memory 256m, --cpus 0.5. E2BSandboxDispatcher calls the E2B REST API
  using E2B_API_KEY from env. Modal and Playwright dispatchers are stub implementations
  that return a SandboxOutput with stdout "back-end not configured" and exit code 1 unless
  the respective SDK is detected on the classpath.

- 1 View ExecutionView with row type ExecutionRow (mirrors Execution). Table updater
  consumes ExecutionEntity events. ONE query getAllExecutions: SELECT * AS executions FROM
  execution_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * ExecutionEndpoint at /api with POST /executions (body
    {taskDescription, backend, wallClockBudgetSeconds, cpuBudgetSeconds, submittedBy};
    mints executionId; calls ExecutionEntity.submit; starts ExecutionWorkflow; returns
    {executionId}), GET /executions (list from getAllExecutions, sorted newest-first),
    GET /executions/{id} (one row), GET /executions/sse (Server-Sent Events forwarded from
    the view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ExecutionTasks.java declaring one Task<R> constant: EXECUTE_TASK = Task.name("Execute task")
  .description("Generate Python to solve the task description, call execute_code, and return
  an ExecutionResult").resultConformsTo(ExecutionResult.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ExecutionRequest, GeneratedCode, ScreeningOutcome, SandboxOutput, HaltReason,
  ExecutionResult, Execution, SandboxBackend, ScreeningDecision, ExecutionOutcome,
  ExecutionStatus, SandboxConfig.

- CodeScreeningGuardrail.java implementing the before-tool-call hook. Extracts the `code`
  parameter from the candidate tool call. Runs 5 pattern checks: (1) host-path traversal
  (regex /etc/|/proc/|~/\.ssh/|C:\\\\Windows), (2) outbound socket creation (socket.connect
  to non-localhost), (3) environment variable harvesting (os\.environ|subprocess.*env),
  (4) dynamic code execution (eval\(|exec\(|__import__\(|compile\(), (5) shell injection
  (os\.system|subprocess\.call|subprocess\.Popen). On any match returns
  Guardrail.reject(patternName) with a structured description. On pass returns
  Guardrail.pass().

- SafetyHaltMonitor.java (Consumer) — implements the resource polling loop per active
  execution. Each running execution gets a scheduled task via
  ScheduledExecutorService.scheduleAtFixedRate(2s). On breach: kill sandbox, call
  ExecutionEntity.halt, cancel the scheduled task.

- SandboxDispatcher.java interface + DockerSandboxDispatcher.java (default). Docker dispatcher
  builds a docker run command string, runs it via ProcessBuilder with the generated code
  piped to stdin, collects stdout/stderr, measures wall-clock and cpu (via /proc/<pid>/stat
  on Linux or OS MXBean on Windows), and returns SandboxOutput.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9103 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-tasks.jsonl with 3 seeded tasks: (1) a data-analysis
  task (compute mean/median/std of a hardcoded numeric dataset, print a summary), (2) a
  file-transformation task (read a CSV string, filter rows where value > threshold, output
  filtered rows), (3) a web-screenshot task (open a URL in a headless browser, take a
  screenshot — usable only with the Playwright back-end; with Docker it gracefully degrades
  to printing the URL).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root pre-filled for the dev-code domain with
  data.data_classes.pii = false (agent never sees user PII; code is the only payload),
  decisions.authority_level = fully-autonomous (the sandbox executes without human approval),
  oversight.human_in_loop = false, oversight.human_on_loop = true (operator monitors the
  live list), failure.failure_modes including "malicious-code-generation",
  "sandbox-escape", "resource-exhaustion", "env-secret-harvest"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/code-execution-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Sandboxed Code Execution", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of execution cards; right = selected-execution detail with task
  description, sandbox config, generated-code block, screening outcome, stdout/stderr,
  generated files, halt reason). Browser title exactly:
  <title>Akka Sample: Sandboxed Code Execution</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(executionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    execute-task.json — 6 ExecutionResult entries covering all ExecutionOutcome values.
      Each entry has a SandboxOutput with stdout (a plausible computation result or script
      output), exitCode 0 or 1, wallClockMs and cpuMs within budget. Plus 2 entries where
      the agent's generated code contains a forbidden pattern (env-harvest and path-traversal)
      so the guardrail blocks them, exercises the retry path. The mock selects a blocked
      entry on the FIRST iteration of every 3rd execution (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(executionId) helper makes per-execution selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeExecutionAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ExecutionTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (screenStep 60s, runStep
  max(wallClockBudgetSeconds+10, 15)s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Execution row record is Optional<T>.
- Lesson 7: ExecutionTasks.java with EXECUTE_TASK = Task.name(...).description(...)
  .resultConformsTo(ExecutionResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9103 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box (Docker back-end)" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeExecutionAgent).
  SafetyHaltMonitor and SandboxRouter are Consumers — they do not make LLM calls.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external wrapper. It fires inside the agent loop before each execute_code call.
- Generated code is passed to the sandbox as the tool parameter, never inlined into the
  agent's instruction text or logged to a file.
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
