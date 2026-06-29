# SPEC — orchestrator-web-file-code

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Generalist Multi-Agent.
**One-line pitch:** Submit a task; an Orchestrator plans the work on a task ledger, dispatches each step to one of four specialist agents (web, file, code, terminal), records the outcome on a progress ledger, and replans when steps fail.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with multiple specialist executors. The Orchestrator owns two ledgers — a **task ledger** (facts known, facts to find, plan, current dispatch) and a **progress ledger** (each subtask's attempt count, verdict, blockers, observed result). Each loop iteration the Orchestrator reads both ledgers, picks the next specialist, and either continues, replans, halts, or completes. On three consecutive failures of the same subtask, or two consecutive replans without progress, the Orchestrator emits a terminal failure.

The blueprint also demonstrates four governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each specialist invocation against an allow-list and a content policy,
- an **automatic safety halt** that trips when the progress ledger detects an unsafe-action signal,
- a **deployer runtime-monitoring** surface (operator dashboard + manual halt control) that lets a human-on-the-loop pause new dispatches without killing in-flight subtasks,
- a **secret sanitizer** that scrubs API-key-shaped strings, JWTs, and obvious credentials from every specialist output before it lands on the progress ledger.

## 3. User-facing flows

The user opens the App UI tab and submits a task via the form.

1. The system creates a `Task` record in `PLANNING` and starts a `TaskWorkflow`.
2. The Orchestrator drafts a `TaskLedger { facts, missing, plan, dispatch }` and emits `TaskPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Orchestrator reads both ledgers and proposes a `DispatchDecision { specialist, subtask, rationale }`.
   - The **before-tool-call guardrail** vets the decision; on rejection the workflow records a `SubtaskBlocked` entry on the progress ledger and asks the Orchestrator to revise.
   - The chosen specialist runs the subtask and returns a typed `SubtaskResult`.
   - The **secret sanitizer** scrubs the result.
   - The workflow appends a `ProgressEntry { specialist, subtask, attempt, verdict, scrubbedResult }` to the progress ledger.
   - The **automatic safety halt** evaluator inspects the new entry; on an unsafe-action signal it emits `TaskHaltedAutomatic` and the workflow ends.
4. The Orchestrator decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After three consecutive failures on the same subtask or two consecutive replans without progress, the Orchestrator emits `FAIL`.
5. On `COMPLETE`, the Orchestrator produces a `TaskAnswer { summary, evidence }` and emits `TaskCompleted`. The Task moves to `COMPLETED`.
6. The operator can press **Halt new dispatches** in the dashboard at any time. The workflow finishes the in-flight subtask, then ends with `TaskHaltedOperator`. The Task moves to `HALTED`.

A `RequestSimulator` (TimedAction) drips a sample task every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OrchestratorAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains task ledger; reads progress ledger. Produces `TaskAnswer` on completion. | `TaskWorkflow` | returns typed result to workflow |
| `WebSurferAgent` | `AutonomousAgent` | Answers web-search subtasks from seeded fixtures (`sample-data/web-fixtures.jsonl`). | `TaskWorkflow` | — |
| `FileSurferAgent` | `AutonomousAgent` | Inspects fixture files (`sample-data/files/*`) and returns excerpts. | `TaskWorkflow` | — |
| `CoderAgent` | `AutonomousAgent` | Drafts or edits code; returns a diff-like `CodeChange`. | `TaskWorkflow` | — |
| `TerminalAgent` | `AutonomousAgent` | Returns simulated command output for an allow-listed command set. | `TaskWorkflow` | — |
| `TaskWorkflow` | `Workflow` | Drives the plan → dispatch-guarded → execute → sanitize → record → decide loop, plus replan and halt branches. | `TaskEndpoint`, `TaskRequestConsumer` | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Holds the task's lifecycle, task ledger, progress ledger, and final answer. | `TaskWorkflow` | `TaskView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `TaskEndpoint` (operator action) | `TaskWorkflow` (polls) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted tasks. | `TaskEndpoint`, `RequestSimulator` | `TaskRequestConsumer` |
| `TaskView` | `View` | List-of-tasks read model for the UI. | `TaskEntity` events | `TaskEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `TaskWorkflow` per submission. | `RequestQueue` events | `TaskWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/task-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, marks any task stuck in `EXECUTING` past 5 minutes as `STUCK`. The workflow polls this and ends with `TaskFailedTimeout`. | scheduler | `TaskEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE, operator halt. | — | `TaskView`, `RequestQueue`, `TaskEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record TaskRequest(String prompt, String requestedBy) {}

record TaskLedger(
    List<String> facts,
    List<String> missing,
    List<String> plan,
    Optional<DispatchDecision> currentDispatch
) {}

record DispatchDecision(
    SpecialistKind specialist,
    String subtask,
    String rationale
) {}

record SubtaskResult(
    SpecialistKind specialist,
    String subtask,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record ProgressEntry(
    int attempt,
    SpecialistKind specialist,
    String subtask,
    ProgressVerdict verdict,
    String scrubbedResult,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ProgressLedger(List<ProgressEntry> entries) {}

record TaskAnswer(String summary, List<String> evidence, Instant producedAt) {}

record Task(
    String taskId,
    String prompt,
    TaskStatus status,
    Optional<TaskLedger> ledger,
    Optional<ProgressLedger> progress,
    Optional<TaskAnswer> answer,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SpecialistKind { WEB, FILE, CODER, TERMINAL }
enum ProgressVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum TaskStatus { PLANNING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`TaskEntity`)

`TaskCreated`, `TaskPlanned`, `SubtaskDispatched`, `SubtaskBlocked`, `SubtaskRecorded`, `LedgerRevised`, `TaskCompleted`, `TaskFailed`, `TaskHaltedAutomatic`, `TaskHaltedOperator`, `TaskFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`TaskSubmitted { taskId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ prompt, requestedBy? }` → `202 { taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=...`.
- `GET /api/tasks/{id}` — one task (full ledgers + answer).
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Generalist Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task, operator halt/resume control, live list of tasks with status pills, expand-row to see the task ledger, the progress ledger entries, and the final answer.

Browser title: `<title>Akka Sample: Generalist Multi-Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `OrchestratorAgent`): every `DispatchDecision` is checked against (a) the specialist allow-list, (b) a content policy that forbids destructive shell commands, file writes outside `/workspace/`, and outbound calls to non-allow-listed hosts. Blocking. Failure → `SubtaskBlocked` entry + replan request.
- **HT1 — automatic safety halt** (`halt`, flavor `automatic-safety-halt`): after each `ProgressEntry` is appended, the workflow runs a deterministic evaluator over the entry's text + the specialist + the subtask. On an unsafe-action signal (e.g., terminal returns evidence of attempted destructive action; coder produces a diff touching forbidden paths), the workflow emits `TaskHaltedAutomatic` and ends.
- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): an operator dashboard pane shows every in-flight task, its current dispatch, and its last progress entry. Two buttons — Halt new dispatches, Resume — drive `SystemControlEntity`. The workflow polls `SystemControlEntity` before each dispatch and exits with `TaskHaltedOperator` if the flag is set.
- **S1 — secret sanitizer** (`sanitizer`, flavor `secret`): every `SubtaskResult.content` is scrubbed by a deterministic redactor that matches AWS access keys, GitHub tokens, JWTs, common API-key patterns, and high-entropy strings of length ≥ 32. The scrubbed result is what lands on the progress ledger and what the Orchestrator sees on the next loop tick.

## 9. Agent prompts

- `OrchestratorAgent` → `prompts/orchestrator.md`. Maintains both ledgers; decides next step.
- `WebSurferAgent` → `prompts/web-surfer.md`. Returns search-style answers from fixtures.
- `FileSurferAgent` → `prompts/file-surfer.md`. Returns file excerpts.
- `CoderAgent` → `prompts/coder.md`. Returns code changes.
- `TerminalAgent` → `prompts/terminal.md`. Returns simulated command output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Find the latest Akka release notes and summarise the new agent features." Task progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a task ledger with a non-empty plan, a progress ledger with 3–8 entries spanning at least Web and File specialists, and a non-empty `TaskAnswer`.
2. **J2** — Submit a task whose plan would invoke a terminal command outside the allow-list ("Run `rm -rf /tmp/*`"). The guardrail blocks the dispatch on the first attempt; the orchestrator replans; the task either completes via a different path or fails after the replan budget is exhausted.
3. **J3** — Submit a task and click **Halt new dispatches** while it is `EXECUTING`. The in-flight subtask finishes; no further dispatches occur; the task ends in `HALTED`.
4. **J4** — Submit a task whose plan exercises the Coder; one fixture response contains an `AKIA...` key shape. The progress ledger entry shows the key replaced by `[REDACTED:aws-access-key]`; the Orchestrator's next prompt never contains the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named orchestrator-web-file-code demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact orchestrator-web-file-code. Java
package io.akka.samples.orchestratorwebfilecode. Akka 3.6.0. HTTP port 9672.

Components to wire (exactly):
- 5 AutonomousAgents:
  * OrchestratorAgent — definition() with two capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_ANSWER).maxIterationsPerTask(2)).
    System prompt from prompts/orchestrator.md. PLAN returns TaskLedger.
    DECIDE returns a NextStep tagged union (DispatchDecision | Replan |
    Complete | Fail). COMPOSE_ANSWER returns TaskAnswer.
  * WebSurferAgent — capability(TaskAcceptance.of(WEB_SEARCH).maxIterationsPerTask(2)).
    Prompt from prompts/web-surfer.md. Returns SubtaskResult.
  * FileSurferAgent — capability(TaskAcceptance.of(FILE_READ).maxIterationsPerTask(2)).
    Prompt from prompts/file-surfer.md. Returns SubtaskResult.
  * CoderAgent — capability(TaskAcceptance.of(WRITE_CODE).maxIterationsPerTask(2)).
    Prompt from prompts/coder.md. Returns SubtaskResult with content holding a
    diff-shaped string.
  * TerminalAgent — capability(TaskAcceptance.of(RUN_COMMAND).maxIterationsPerTask(2)).
    Prompt from prompts/terminal.md. Returns SubtaskResult.

- 1 Workflow TaskWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  dispatchStep -> sanitizeStep -> recordStep -> autoHaltEvalStep -> decideStep
  -> [back to checkHaltStep or to completeStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120) (covers any specialist call), decideStep ofSeconds(45),
    completeStep ofSeconds(60). defaultStepRecovery(maxRetries(2).failoverTo
    (TaskWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits TaskHaltedOperator on TaskEntity).
  guardrailStep runs the deterministic vetter over DispatchDecision; on reject
  records a SubtaskBlocked entry via TaskEntity.recordBlock(subtask, reason)
  and loops back to proposeStep.
  dispatchStep uses switch on DispatchDecision.specialist to call the
  matching specialist agent via forAutonomousAgent(...).runSingleTask(...)
  then forTask(taskId).result(...).
  sanitizeStep applies SecretScrubber.scrub to the SubtaskResult.content.
  recordStep calls TaskEntity.recordProgress(entry).
  autoHaltEvalStep runs SafetyEvaluator.evaluate(entry) and, on unsafe,
  transitions to haltedStep (emits TaskHaltedAutomatic).
  decideStep calls forAutonomousAgent(OrchestratorAgent.class, DECIDE);
  on Continue or Replan loops; on Complete transitions to composeAnswerStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity TaskEntity holding Task state. emptyState() returns
  Task.initial("", null) with no commandContext() reference. Commands:
  createTask, recordPlan, recordDispatch, recordBlock, recordProgress,
  reviseLedger, completeTask, failTask, haltAutomatic, haltOperator,
  timeoutFail, getTask. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueueTask(taskId, prompt,
  requestedBy) emitting TaskSubmitted.

- 1 View TaskView with row type TaskRow (mirror of Task minus heavy ledger
  payloads — truncate to last 3 progress entries plus counts; the UI fetches
  the full task by id on click). Table updater consumes TaskEntity events.
  ONE query getAllTasks SELECT * AS tasks FROM task_view. No WHERE status
  filter — caller filters client-side (Lesson 2).

- 1 Consumer TaskRequestConsumer subscribed to RequestQueue events; on
  TaskSubmitted starts a TaskWorkflow with taskId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/task-prompts.jsonl and calls
    RequestQueue.enqueueTask.
  * StuckTaskMonitor — every 30s, queries TaskView.getAllTasks, filters
    EXECUTING tasks whose createdAt is older than 5 minutes, calls
    TaskEntity.timeoutFail; TaskWorkflow polls TaskEntity.getTask in its
    decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks (filters client-side
    from getAllTasks), GET /tasks/{id}, GET /tasks/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- OrchestratorTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo TaskLedger), DECIDE (NextStep), COMPOSE_ANSWER (TaskAnswer).
- SpecialistTasks.java declaring four Task<R> constants: WEB_SEARCH,
  FILE_READ, WRITE_CODE, RUN_COMMAND (all resultConformsTo SubtaskResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits Continue, Replan, Complete, Fail (each carrying its own payload —
  Continue with DispatchDecision, Replan with revisedLedger, Complete with
  TaskAnswer, Fail with failureReason).
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- application/DispatchGuardrail.java — deterministic vetter. Reject if the
  specialist is not WEB/FILE/CODER/TERMINAL, if a TERMINAL subtask matches
  /^(rm|sudo|mkfs|dd|chmod 777|>>?\s*\/(etc|proc|sys|dev))/, if a CODER
  subtask references writes outside /workspace/, if a WEB subtask names a
  host not on the allow-list (akka.io, doc.akka.io, github.com).
- application/SafetyEvaluator.java — deterministic post-execution check.
  Flag UNSAFE on TERMINAL results containing "permission denied" plus the
  word "sudo", on CODER results whose diff touches /etc, /root, or any
  path matching ~/.ssh/, on WEB results whose content includes attempted
  prompt-injection markers (e.g., "ignore previous instructions",
  "<system>"). Otherwise OK.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9672 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-prompts.jsonl with 8 canned task
  prompts spanning web, file, code, and terminal needs.
- src/main/resources/sample-data/web-fixtures.jsonl — 12 canned web
  fixtures (host, path, title, excerpt). Used by WebSurferAgent.
- src/main/resources/sample-data/files/* — 6 short text files used by
  FileSurferAgent. Include one file whose content contains an AKIA-shaped
  key fragment for the J4 acceptance test (the agent must surface it; the
  sanitizer must scrub it).
- src/main/resources/sample-data/commands.jsonl — allow-listed terminal
  commands with canned output (ls, pwd, cat <fixture path>, echo …, head,
  wc -l).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls (G1, HT1, HO1, S1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/orchestrator.md, prompts/web-surfer.md, prompts/file-surfer.md,
  prompts/coder.md, prompts/terminal.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Generalist
  Multi-Agent", one-line pitch, prerequisites (including the integration
  form's host-software requirement: None), generate-the-system, what-you-
  get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for ledgers and answer). Browser title
  exactly: <title>Akka Sample: Generalist Multi-Agent</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class
  name and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: orchestrator.json, web-surfer.json, file-surfer.json,
  coder.json, terminal.json), picks one entry pseudo-randomly per call, and
  deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    orchestrator.json — a sectioned file with three lists keyed by task id:
      "PLAN" → 4–6 TaskLedger entries (facts, missing, plan steps that span
      web/file/code/terminal).
      "DECIDE" → 4–6 NextStep entries covering Continue (with
      DispatchDecision across all four specialists), Replan, Complete, Fail.
      The Continue entries advance through a plausible task narrative when
      iterated in order.
      "COMPOSE_ANSWER" → 4–6 TaskAnswer entries with 60–120 word summaries
      and 3–5 evidence bullets.
    web-surfer.json — 6 SubtaskResult entries, ok=true, content fields are
      mocked search results (4–6 lines each).
    file-surfer.json — 6 SubtaskResult entries; ONE entry's content must
      include the literal substring "AKIAIOSFODNN7EXAMPLE" so the J4
      sanitizer test fires.
    coder.json — 5 SubtaskResult entries with diff-shaped content; one
      entry's content includes a "Bearer eyJhbGciOi..."-shaped token so the
      sanitizer test is exercised end-to-end.
    terminal.json — 5 SubtaskResult entries; outputs aligned with the
      allow-listed command set.
- A MockModelProvider.seedFor(taskId) helper makes the selection
  deterministic per task id so the same task in dev produces the same
  output across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent — the
  base class clause must read `extends AutonomousAgent` for every agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, dispatchStep,
  decideStep, completeStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the Task entity state (ledger, progress, answer, failureReason,
  haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion OrchestratorTasks.java and
  SpecialistTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9672 in application.conf — picked from the
  available range.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the
  box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM — delete
  removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
