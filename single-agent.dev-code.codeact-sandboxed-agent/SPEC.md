# SPEC — codeact-sandboxed-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CodeAct Agent.
**One-line pitch:** A user submits a natural-language coding task; one AI agent writes code, runs it inside a sandboxed executor, inspects the output, and iterates — halting automatically if destructive or secret-leaking behaviour is detected — until the task is solved or the iteration budget is exhausted.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `CodeActAgent` (AutonomousAgent) carries the entire code-generation-and-execution loop; the surrounding components only validate its inputs, gate its tool calls, and sanitize its outputs. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs inside `SandboxGuardrail` every time the agent emits an `execute_code` tool call — inspecting the code for forbidden syscalls, disallowed imports, and resource-abuse patterns before a single byte of generated code reaches the executor.
- An **automatic safety halt** in `SafetyHaltMonitor` watches execution outputs in real time; if it detects a destructive action signature or a secret-like token in the output stream, it emits a `HaltTriggered` event and the workflow transitions the task to `HALTED` immediately, regardless of which iteration the agent is on.
- A **secret sanitizer** runs inside a `SecretSanitizer` Consumer after every `CodeExecuted` event, scrubbing API keys, credential patterns, and high-entropy tokens from execution outputs before they land in the event log or the UI.

The blueprint demonstrates that the single-agent pattern is compatible with high-risk tooling — three independent gates sit around the one decision-making LLM call without requiring a second agent.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **task** from the dropdown (three seeded tasks: a CSV-to-JSON data transform, a prime-sieve algorithm, a word-frequency counter) or types a custom natural-language task description.
2. The user optionally provides **context data** — a small inline text blob the generated code may read as standard input.
3. The user clicks **Run task**. The UI POSTs to `/api/tasks` and receives a `taskId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s the workflow starts and the card transitions to `EXECUTING`.
5. Within ~10–30 s, the agent's code-generation-and-execution loop runs its first iteration. The card transitions through `EXECUTING` → either another `EXECUTING` iteration (if the output was wrong and the agent retries) or `SOLVED` (if the output matches the acceptance criterion). Each iteration's generated code snippet and execution output appear in an expandable log on the card.
6. If the safety halt fires at any point, the card transitions immediately to `HALTED` with the halt reason visible.
7. On `SOLVED`, the final accepted output appears prominently on the card. The iteration count and total wall-clock time are shown.
8. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TaskEntity`, `TaskView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TaskEntity` | `EventSourcedEntity` | Per-task lifecycle: submitted → executing → solved / halted / failed. Source of truth. | `TaskEndpoint`, `SecretSanitizer`, `ExecutionWorkflow` | `TaskView` |
| `SecretSanitizer` | `Consumer` | Subscribes to `CodeExecuted` events; scrubs secrets from outputs; calls `TaskEntity.attachSanitizedOutput`. | `TaskEntity` events | `TaskEntity` |
| `ExecutionWorkflow` | `Workflow` | One workflow per task. Steps: `executeStep` → `inspectStep` → loop or `solvedStep`. | started by `TaskEndpoint` after submit | `CodeActAgent`, `TaskEntity` |
| `CodeActAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task definition and context data as a task attachment; emits `execute_code` tool calls; returns `TaskResolution`. | invoked by `ExecutionWorkflow` | returns resolution |
| `SandboxGuardrail` | supporting class | `before-tool-call` hook registered on `CodeActAgent`; rejects code with forbidden patterns. | wired into `CodeActAgent` | — |
| `SafetyHaltMonitor` | supporting class | Inspects execution outputs for destructive or secret-like patterns; emits `HaltTriggered` if matched. | called by `ExecutionWorkflow.executeStep` | `TaskEntity` |
| `TaskView` | `View` | Read model: one row per task for the UI. | `TaskEntity` events | `TaskEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TaskDefinition(
    String taskId,
    String description,
    String contextData,        // may be empty; passed as stdin to generated code
    String acceptanceCriterion,
    String submittedBy,
    Instant submittedAt
) {}

record CodeSnippet(
    int iterationNumber,
    String language,           // e.g. "python", "javascript"
    String code,
    Instant generatedAt
) {}

record ExecutionResult(
    int iterationNumber,
    String rawOutput,
    boolean halted,
    String haltReason,         // null when halted=false
    Instant executedAt
) {}

record SanitizedOutput(
    int iterationNumber,
    String sanitizedOutput,
    List<String> secretCategoriesFound   // e.g. ["aws-key","github-token"]
) {}

record TaskResolution(
    ResolutionStatus status,
    String finalOutput,
    int iterationsUsed,
    Instant resolvedAt
) {}
enum ResolutionStatus { SOLVED, FAILED, HALTED }

record Task(
    String taskId,
    Optional<TaskDefinition> definition,
    List<CodeSnippet> codeHistory,
    List<SanitizedOutput> outputHistory,
    Optional<TaskResolution> resolution,
    TaskStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus {
    SUBMITTED, EXECUTING, SOLVED, HALTED, FAILED
}
```

Events on `TaskEntity`: `TaskSubmitted`, `CodeExecuted`, `OutputSanitized`, `HaltTriggered`, `TaskSolved`, `TaskFailed`.

Every nullable lifecycle field on the `Task` row record is `Optional<T>` (Lesson 6). The `codeHistory` and `outputHistory` lists start as `List.of()` and accumulate one entry per iteration.

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tasks` — body `{ description, contextData, acceptanceCriterion, submittedBy }` → `{ taskId }`.
- `GET /api/tasks` — list all tasks, newest-first.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: CodeAct Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tasks (status pill + resolution badge + age) and a right pane with the selected task's detail — task description, context data, per-iteration code snippets and sanitized outputs (expandable log), resolution summary, and iteration count.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`, applied inside `SandboxGuardrail`): inspects every `execute_code` tool call the agent emits. Checks for: forbidden syscalls (`os.system`, `subprocess`, `exec`, `eval` outside the sandbox API), disallowed import patterns (network libraries, filesystem writers), and code that exceeds a configurable line-length limit (a proxy for obfuscated one-liners). On rejection returns a structured error to the agent loop naming the forbidden pattern; the agent loop counts the iteration and rewrites the code. `maxIterationsPerTask(5)` caps total attempts.
- **H1 — automatic safety halt** (`halt`, `automatic-safety-halt`, applied inside `SafetyHaltMonitor`): called synchronously by `ExecutionWorkflow.executeStep` after each sandbox run. Scans the raw execution output for destructive action signatures (file-deletion markers, fork-bomb indicators, outbound-network tokens) and secret-like patterns (AWS key prefixes, PEM headers, high-entropy 40-char alphanum strings). On match emits `HaltTriggered` and transitions the task to `HALTED` without starting another iteration.
- **S1 — secret sanitizer** (`sanitizer`, `secret`, applied inside `SecretSanitizer` Consumer): subscribes to `CodeExecuted` events. Redacts API keys, bearer tokens, credential assignment patterns, and high-entropy tokens from the raw execution output before the sanitized form is written to the entity and displayed in the UI. The raw output is preserved on the entity for audit; the UI shows only the sanitized form.

## 9. Agent prompts

- `CodeActAgent` → `prompts/codeact-agent.md`. The single decision-making LLM. System prompt instructs it to read the task description and context data, write executable code, call the `execute_code` tool with the code, inspect the output, and iterate until the acceptance criterion is met or the budget is exhausted.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the CSV-to-JSON seed task; within 30 s the task transitions to `SOLVED` with the correct JSON output visible and an iteration count of 1–3.
2. **J2** — The agent generates code containing `import subprocess` on its first iteration; the `before-tool-call` guardrail rejects it with a `forbidden-import` error; the agent rewrites without the import; the second iteration runs cleanly and solves the task.
3. **J3** — An execution output containing the string `GITHUB_TOKEN=ghp_xxxxx` triggers the safety halt; the task transitions to `HALTED`; the raw output never appears in the UI; the halt reason names the matched pattern.
4. **J4** — An execution output containing `AWS_SECRET_ACCESS_KEY=AKIA...` passes the halt check (not a destructive action) but is caught by the secret sanitizer; the entity log shows `[REDACTED-AWS-KEY]` in place of the credential; the raw form is on the entity for audit but never surfaces in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named codeact-sandboxed-agent demonstrating the single-agent × dev-code cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-codeact-sandboxed-agent. Java package io.akka.samples.codeactagent.
Akka 3.6.0. HTTP port 9337.

Components to wire (exactly):

- 1 AutonomousAgent CodeActAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/codeact-agent.md>) and
  .capability(TaskAcceptance.of(EXECUTE_CODE_TASK).maxIterationsPerTask(5)).
  The task receives the natural-language task description and acceptance criterion as its
  instruction text, and the context data as a task ATTACHMENT (NOT as inline prompt text —
  Akka's TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  TaskResolution{status: ResolutionStatus (SOLVED/FAILED/HALTED), finalOutput: String,
  iterationsUsed: int, resolvedAt: Instant}. The agent is configured with a before-tool-call
  guardrail (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop retries the code within its 5-iteration budget.

- 1 Workflow ExecutionWorkflow per taskId with three logical steps:
  * executeStep — emits a CodeSnippet by calling the agent's execute_code tool call (routed
    through SandboxGuardrail). After execution, calls SafetyHaltMonitor.inspect(rawOutput);
    if halt detected, calls TaskEntity.triggerHalt(reason) and transitions workflow to
    haltedStep. Otherwise calls TaskEntity.recordExecution(snippet, rawOutput).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ExecutionWorkflow::error).
  * inspectStep — fetches the latest SanitizedOutput from the entity; if the output
    satisfies the acceptance criterion (checked by a rule-based AcceptanceChecker), calls
    TaskEntity.markSolved(resolution) and transitions to solvedStep. Otherwise, if
    iterationsUsed < maxIterations, loops back to executeStep. If budget exhausted, calls
    TaskEntity.markFailed("iteration budget exhausted"). WorkflowSettings.stepTimeout 15s.
  * solvedStep and haltedStep — terminal; call entity to set finishedAt.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TaskEntity (one per taskId). State Task{taskId: String,
  definition: Optional<TaskDefinition>, codeHistory: List<CodeSnippet>,
  outputHistory: List<SanitizedOutput>, resolution: Optional<TaskResolution>, status: TaskStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. TaskStatus enum: SUBMITTED, EXECUTING,
  SOLVED, HALTED, FAILED. Events: TaskSubmitted{definition}, CodeExecuted{snippet, rawOutput},
  OutputSanitized{sanitizedOutput}, HaltTriggered{reason}, TaskSolved{resolution},
  TaskFailed{reason}. Commands: submit, recordExecution, attachSanitizedOutput, triggerHalt,
  markSolved, markFailed, getTask. emptyState() returns Task.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier. codeHistory and outputHistory
  are accumulated across events — each applier appends to the existing list.

- 1 Consumer SecretSanitizer subscribed to TaskEntity events; on CodeExecuted runs a
  regex+heuristic pipeline (AWS key patterns, GitHub tokens, PEM headers, bearer token
  patterns, high-entropy alphanum strings ≥32 chars, credential-assignment patterns
  like KEY=<value>) over rawOutput, computes the list of secret categories found, builds
  SanitizedOutput, then calls TaskEntity.attachSanitizedOutput(sanitizedOutput).

- 1 View TaskView with row type TaskRow (mirrors Task minus any rawOutput fields — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes TaskEntity events. ONE query getAllTasks: SELECT * AS tasks FROM task_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks (body {description, contextData,
    acceptanceCriterion, submittedBy}; mints taskId; calls TaskEntity.submit; returns
    {taskId}), GET /tasks (list from getAllTasks, sorted newest-first), GET /tasks/{id}
    (one row), GET /tasks/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Supporting classes:

- CodeActTasks.java declaring one Task<R> constant: EXECUTE_CODE_TASK = Task
  .name("Execute code task")
  .description("Write and execute code to solve the submitted task; iterate on the output until the acceptance criterion is met")
  .resultConformsTo(TaskResolution.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records TaskDefinition, CodeSnippet, ExecutionResult, SanitizedOutput,
  TaskResolution, ResolutionStatus, Task, TaskStatus.

- SandboxGuardrail.java implementing the before-tool-call hook. Inspects the code argument
  of each execute_code tool call for: forbidden imports (subprocess, socket, ftplib,
  paramiko, smtplib), forbidden built-ins (eval, exec, __import__, compile), filesystem
  write patterns (open(..., 'w'), os.remove, shutil.rmtree), and lines exceeding 400
  characters (obfuscation proxy). On any match returns Guardrail.reject(<structured-error>)
  naming the forbidden pattern; the agent loop retries. Passing calls flow to the sandbox.

- SafetyHaltMonitor.java — pure deterministic logic (no LLM). Inputs: raw execution
  output String. Outputs: Optional<String> haltReason. Checks: destructive markers
  (strings like "Deleted", "Removed", "rm -rf", "DROP TABLE", outbound-URL patterns),
  fork-bomb indicators (rapid repeated output), PEM header ("-----BEGIN"), and high-entropy
  secret-like tokens (≥40 char alphanum matching AWS/GCP/GitHub key prefixes). Returns
  Optional.empty() for clean output, Optional.of(reason) to trigger halt.

- AcceptanceChecker.java — pure rule-based logic. Inputs: TaskDefinition and
  SanitizedOutput. Outputs: boolean accepted. For seeded tasks uses exact-match or
  pattern-match on the acceptance criterion string. Deployers may replace with an LLM-based
  evaluator — the interface stays the same.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9337 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CodeActAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-tasks.jsonl with 3 seeded tasks:
  (1) CSV-to-JSON transform (10-row input, acceptance: valid JSON array),
  (2) prime-sieve for primes up to 100 (acceptance: exact list match),
  (3) word-frequency counter (10-word sentence, acceptance: correct top-3 words).
  Each task includes contextData and acceptanceCriterion.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes appropriate for dev-code,
  decisions.authority_level = automated-with-halt, oversight.human_in_loop = false
  (the agent runs autonomously; the safety halt is the primary stop mechanism),
  failure.failure_modes including "runaway-loop", "secret-exfiltration", "forbidden-syscall",
  "incorrect-output-accepted"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/codeact-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: CodeAct Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of task cards; right = selected-task detail with task description, context data,
  per-iteration code snippets and sanitized outputs in an expandable accordion, resolution
  summary, and iteration count). Browser title exactly:
  <title>Akka Sample: CodeAct Agent</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(taskId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    execute-code-task.json — 8 TaskResolution entries covering SOLVED, FAILED, HALTED.
      Each SOLVED entry has a finalOutput matching the seed task's acceptance criterion and
      an iterationsUsed value in the range 1–4. Each FAILED entry has a descriptive
      finalOutput explaining the failure. Each HALTED entry has a finalOutput containing a
      detectable secret-like string (so the halt path is exercised). Plus 2 entries with
      code that contains a forbidden import (so the guardrail path is exercised on the first
      iteration). The mock should select a guardrail-triggering entry on the FIRST iteration
      of every 3rd task (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(taskId) helper makes per-task selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeActAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CodeActTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (executeStep
  60s, inspectStep 15s, solvedStep 5s, haltedStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Task row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: CodeActTasks.java with EXECUTE_CODE_TASK = Task.name(...).description(...)
  .resultConformsTo(TaskResolution.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9337 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeActAgent). The
  AcceptanceChecker and SafetyHaltMonitor are rule-based and do NOT make LLM calls.
- The context data is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated executeStep uses TaskDef.attachment(...) for context
  data.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent's tool call returns.
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
