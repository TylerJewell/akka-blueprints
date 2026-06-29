# SPEC — code-assistant-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Agentic Code Assistant.
**One-line pitch:** Submit a coding task; a Planner drafts a change plan on a plan ledger, dispatches each step to a Reader, Editor, or Runner agent, records results on an edit log, and replans or stops when tests fail or a loop bound is reached.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to an autonomous code-editing loop. The Planner owns two ledgers — a **plan ledger** (files to read, steps to execute, current dispatch) and an **edit log** (each read/edit/run attempt with its outcome and diff). Each loop iteration the Planner reads both ledgers, picks the next action (read a file, apply an edit, run tests), and either continues, replans, commits, or fails.

The blueprint also demonstrates three governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each edit or run dispatch against path policy and command allow-lists before any file is touched,
- a **ci-gate** that checks the test-run result after every runner call and blocks commit if tests are red — forcing the planner to revise or exhaust its budget,
- an **automatic safety halt** that trips on loop-bound detection (too many consecutive failures or replans) so the session cannot spin indefinitely.

## 3. User-facing flows

The user opens the App UI tab and submits a coding task via the form.

1. The system creates a `Session` record in `PLANNING` and starts a `SessionWorkflow`.
2. The Planner drafts a `PlanLedger { targetFiles, steps, currentDispatch }` and emits `SessionPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads both ledgers and proposes an `ActionDecision { actionKind, targetFile, instruction, rationale }`.
   - The **before-tool-call guardrail** vets the decision; on rejection the workflow records an `EditBlocked` entry on the edit log and asks the Planner to revise.
   - The chosen agent (Reader, Editor, or Runner) executes the action and returns a typed `ActionResult`.
   - The workflow appends an `EditEntry { actionKind, targetFile, attempt, verdict, diff, testOutput, recordedAt }` to the edit log.
   - If the action was `RUN_TESTS`, the **ci-gate** evaluates the test output; on failure it records a `CiGateFailed` entry and the Planner must revise.
   - The **automatic safety halt** evaluator checks consecutive failure counts; on the loop-bound threshold it emits `SessionHaltedAutomatic` and the workflow ends.
4. The Planner decides each tick: `CONTINUE`, `REPLAN`, `COMMIT`, or `FAIL`. After three consecutive test failures or two consecutive replans without progress, the Planner emits `FAIL`.
5. On `COMMIT`, the Planner produces a `CommitSummary { message, filesChanged, testsPassed }` and emits `SessionCommitted`. The Session moves to `COMMITTED`.
6. The operator can press **Halt new edits** in the dashboard at any time. The workflow finishes the in-flight action, then ends with `SessionHaltedOperator`. The Session moves to `HALTED`.

A `TaskSimulator` (TimedAction) drips a sample coding task every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Drafts the change plan; decides next action each loop tick; produces `CommitSummary` on completion. | `SessionWorkflow` | returns typed result to workflow |
| `ReaderAgent` | `AutonomousAgent` | Returns file contents from seeded fixtures (`sample-data/files/*`). | `SessionWorkflow` | — |
| `EditorAgent` | `AutonomousAgent` | Produces a unified diff for a target file within `/workspace/`. | `SessionWorkflow` | — |
| `RunnerAgent` | `AutonomousAgent` | Returns simulated test-run output for an allow-listed test command set. | `SessionWorkflow` | — |
| `SessionWorkflow` | `Workflow` | Drives the plan → read/edit/run → guardrail → execute → ci-gate → record → decide loop, plus replan and halt branches. | `SessionEndpoint`, `TaskRequestConsumer` | `SessionEntity` |
| `SessionEntity` | `EventSourcedEntity` | Holds the session lifecycle, plan ledger, edit log, and final commit summary. | `SessionWorkflow` | `SessionView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `SessionEndpoint` (operator action) | `SessionWorkflow` (polls) |
| `TaskQueue` | `EventSourcedEntity` | Audit log of submitted tasks. | `SessionEndpoint`, `TaskSimulator` | `TaskRequestConsumer` |
| `SessionView` | `View` | List-of-sessions read model for the UI. | `SessionEntity` events | `SessionEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Subscribes to `TaskQueue` events; starts a `SessionWorkflow` per submission. | `TaskQueue` events | `SessionWorkflow` |
| `TaskSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/task-prompts.jsonl` and enqueues it. | scheduler | `TaskQueue` |
| `StuckSessionMonitor` | `TimedAction` | Every 30 s, marks any session stuck in `EXECUTING` past 5 minutes as `STUCK`. The workflow polls this and ends with `SessionFailedTimeout`. | scheduler | `SessionEntity` |
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, get, list, SSE, operator halt. | — | `SessionView`, `TaskQueue`, `SessionEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TaskRequest(String prompt, String requestedBy) {}

record PlanLedger(
    List<String> targetFiles,
    List<String> steps,
    Optional<ActionDecision> currentDispatch
) {}

record ActionDecision(
    ActionKind actionKind,
    String targetFile,
    String instruction,
    String rationale
) {}

record ActionResult(
    ActionKind actionKind,
    String targetFile,
    boolean ok,
    String content,
    Optional<String> diff,
    Optional<String> testOutput,
    Optional<String> errorReason
) {}

record EditEntry(
    int attempt,
    ActionKind actionKind,
    String targetFile,
    String instruction,
    EditVerdict verdict,
    String content,
    Optional<String> diff,
    Optional<String> testOutput,
    Optional<String> blocker,
    Instant recordedAt
) {}

record EditLog(List<EditEntry> entries) {}

record CommitSummary(
    String message,
    List<String> filesChanged,
    boolean testsPassed,
    Instant producedAt
) {}

record Session(
    String sessionId,
    String prompt,
    SessionStatus status,
    Optional<PlanLedger> plan,
    Optional<EditLog> editLog,
    Optional<CommitSummary> commitSummary,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ActionKind { READ_FILE, EDIT_FILE, RUN_TESTS }
enum EditVerdict { OK, BLOCKED_BY_GUARDRAIL, CI_GATE_FAILED, FAILED, UNSAFE }
enum SessionStatus { PLANNING, EXECUTING, COMMITTED, FAILED, HALTED, STUCK }
```

### Events (`SessionEntity`)

`SessionCreated`, `SessionPlanned`, `ActionDispatched`, `EditBlocked`, `CiGateFailed`, `EditRecorded`, `PlanRevised`, `SessionCommitted`, `SessionFailed`, `SessionHaltedAutomatic`, `SessionHaltedOperator`, `SessionFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`TaskQueue`)

`TaskSubmitted { sessionId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ prompt, requestedBy? }` → `202 { sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=...`.
- `GET /api/sessions/{id}` — one session (full plan ledger + edit log + commit summary).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Agentic Code Assistant"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task, operator halt/resume control, live list of sessions with status pills, expand-row to see the plan ledger, edit log entries, and the commit summary.

Browser title: `<title>Akka Sample: Agentic Code Assistant</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `PlannerAgent`): every `ActionDecision` is checked against (a) the action allow-list (`READ_FILE`, `EDIT_FILE`, `RUN_TESTS` only), (b) a path policy that forbids edits outside `/workspace/`, writes to `/etc`, `/root`, or `~/.ssh/`, and (c) a test-command allow-list (`mvn test`, `gradle test`, `npm test`, `pytest`). Blocking. Failure → `EditBlocked` entry + replan request.
- **E1 — CI test gate** (`ci-gate`, flavor `test-gate`): after every `RUN_TESTS` result, the workflow's `ciGateStep` parses the test output for a pass/fail signal. On failure it records a `CiGateFailed` entry and increments the consecutive-failure counter. Three consecutive failures → `FAIL`. This gate runs deterministically, not via the LLM.
- **HT1 — automatic safety halt** (`halt`, flavor `automatic-safety-halt`): after each `EditEntry` is appended, the workflow checks the consecutive-failure and replan counters. On loop-bound breach (three consecutive test failures or two consecutive replans without an `OK` edit in between), the workflow emits `SessionHaltedAutomatic` and ends.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Maintains both ledgers; decides next action.
- `ReaderAgent` → `prompts/reader.md`. Returns file contents from fixtures.
- `EditorAgent` → `prompts/editor.md`. Returns unified diffs.
- `RunnerAgent` → `prompts/runner.md`. Returns simulated test output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Add a `multiply` method to `Calculator.java` and update the unit tests to cover it." Session progresses `PLANNING → EXECUTING → COMMITTED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a plan ledger with a non-empty steps list, an edit log with 3–8 entries including at least one `READ_FILE`, one `EDIT_FILE`, and one `RUN_TESTS`, and a non-empty `CommitSummary`.
2. **J2** — Submit a task whose plan proposes writing to `/etc/hosts`. The guardrail blocks the dispatch on the first attempt; the planner replans; the session either commits via a different path or fails after the replan budget is exhausted. The forbidden write never executes.
3. **J3** — Submit a task that produces a test failure. The CI gate records `CiGateFailed`; the planner revises; if it fails three times the session ends in `FAILED` with a clear `failureReason`.
4. **J4** — Submit a task and click **Halt new edits** while `EXECUTING`. The in-flight action finishes; no further dispatches occur; session ends in `HALTED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named code-assistant-loop demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-code-assistant-loop.
Java package io.akka.samples.agenticcodeassistant. Akka 3.6.0. HTTP port 9802.

Components to wire (exactly):
- 4 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(DRAFT_PLAN).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(DECIDE_ACTION).maxIterationsPerTask(3))
      capability(TaskAcceptance.of(COMPOSE_COMMIT).maxIterationsPerTask(2))
    System prompt from prompts/planner.md. DRAFT_PLAN returns PlanLedger.
    DECIDE_ACTION returns a NextAction tagged union (Continue(ActionDecision)
    | Replan(PlanLedger revised) | Commit(CommitSummary stub) | Fail(reason)).
    COMPOSE_COMMIT returns CommitSummary.
  * ReaderAgent — capability(TaskAcceptance.of(READ_FILE).maxIterationsPerTask(2)).
    Prompt from prompts/reader.md. Returns ActionResult.
  * EditorAgent — capability(TaskAcceptance.of(EDIT_FILE).maxIterationsPerTask(2)).
    Prompt from prompts/editor.md. Returns ActionResult with diff populated.
  * RunnerAgent — capability(TaskAcceptance.of(RUN_TESTS).maxIterationsPerTask(2)).
    Prompt from prompts/runner.md. Returns ActionResult with testOutput populated.

- 1 Workflow SessionWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  dispatchStep -> recordStep -> ciGateStep -> autoHaltEvalStep -> decideStep
  -> [back to checkHaltStep or to composeCommitStep / commitStep / failStep /
  haltedStep].
  Step timeouts (Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120), decideStep ofSeconds(45), commitStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(SessionWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits SessionHaltedOperator on SessionEntity).
  guardrailStep runs EditGuardrail.vet(ActionDecision); on reject records
  EditBlocked entry via SessionEntity.recordBlock and loops back to proposeStep.
  dispatchStep uses switch on ActionDecision.actionKind to call the matching
  agent via forAutonomousAgent(...).runSingleTask(...).
  recordStep calls SessionEntity.recordEdit(entry).
  ciGateStep runs CiGate.evaluate(entry) only when actionKind == RUN_TESTS;
  on failure records CiGateFailed and increments the consecutive-failure
  counter; at threshold transitions to failStep.
  autoHaltEvalStep checks consecutive-failure and replan counters; on
  loop-bound breach transitions to haltedStep (emits SessionHaltedAutomatic).
  decideStep calls forAutonomousAgent(PlannerAgent.class, DECIDE_ACTION);
  on Continue or Replan loops; on Commit transitions to composeCommitStep
  -> commitStep; on Fail transitions to failStep.

- 1 EventSourcedEntity SessionEntity holding Session state. Commands:
  createSession, recordPlan, recordDispatch, recordBlock, recordEdit,
  recordCiGateFail, revisePlan, commitSession, failSession, haltAutomatic,
  haltOperator, timeoutFail, getSession. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity TaskQueue with command enqueueTask(sessionId, prompt,
  requestedBy) emitting TaskSubmitted.

- 1 View SessionView with row type SessionRow (mirror of Session minus heavy
  edit-log payloads — truncate to last 3 edit entries plus counts; the UI
  fetches the full session by id on click). ONE query getAllSessions SELECT *
  AS sessions FROM session_view. No WHERE status filter — caller filters
  client-side (Lesson 2).

- 1 Consumer TaskRequestConsumer subscribed to TaskQueue events; on
  TaskSubmitted starts a SessionWorkflow with sessionId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 90s, reads next line from
    src/main/resources/sample-events/task-prompts.jsonl and calls
    TaskQueue.enqueueTask.
  * StuckSessionMonitor — every 30s, queries SessionView.getAllSessions,
    filters EXECUTING sessions whose createdAt is older than 5 minutes,
    calls SessionEntity.timeoutFail; SessionWorkflow polls
    SessionEntity.getSession in its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions, GET /sessions (client-side
    filter from getAllSessions), GET /sessions/{id}, GET /sessions/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: DRAFT_PLAN
  (resultConformsTo PlanLedger), DECIDE_ACTION (NextAction),
  COMPOSE_COMMIT (CommitSummary).
- AgentTasks.java declaring three Task<R> constants: READ_FILE, EDIT_FILE,
  RUN_TESTS (all resultConformsTo ActionResult).
- Domain records as listed in SPEC §5, plus a NextAction sealed interface
  with permits Continue(ActionDecision), Replan(PlanLedger revised),
  Commit(CommitSummary stub), Fail(String reason).
- application/EditGuardrail.java — deterministic vetter. Reject if actionKind
  is not in {READ_FILE, EDIT_FILE, RUN_TESTS}; if EDIT_FILE targets a path
  outside /workspace/ or matching /^(\\/etc|\\/root|.*\\.ssh)/, or any path
  starting with ~; if RUN_TESTS names a command not in the allow-list (mvn
  test, gradle test, npm test, pytest, go test ./...).
- application/CiGate.java — deterministic test-output parser. Parses for
  BUILD SUCCESS / BUILD FAILURE (Maven), Tests passed / Tests failed
  (pytest/Node), ok / FAIL (Go), passing / failing (generic). Returns
  CiVerdict{passed, totalTests, failedTests, summary}.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9802 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-prompts.jsonl with 8 canned coding
  tasks spanning file reads, single-method edits, test additions, and
  refactors.
- src/main/resources/sample-data/files/* — 8 short Java/Python/config fixture
  files used by ReaderAgent and EditorAgent. Include Calculator.java,
  CalculatorTest.java, utils.py, config.yml, README.md, legacy-config.md,
  build.gradle, pom.xml.
- src/main/resources/sample-data/test-outputs.jsonl — canned test outputs
  for RunnerAgent: 4 PASS results and 2 FAIL results in Maven and pytest
  formats.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint).
- eval-matrix.yaml at the project root with 3 controls (G1, E1, HT1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data, decisions,
  failure, oversight, operations, and compliance for the dev-code domain;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/reader.md, prompts/editor.md, prompts/runner.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Agentic Code Assistant",
  one-line pitch, prerequisites (integration form host-software: None),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-
  mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs: Overview,
  Architecture (4 mermaid diagrams + click-to-expand component table with
  syntax-highlighted Java snippets), Risk Survey (7 sub-tabs with answers
  from risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with status
  pills and expand-on-click for plan ledger, edit log entries, and commit
  summary). Browser title exactly:
  <title>Akka Sample: Agentic Code Assistant</title>. No subtitle on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value passed via MCP tool environment
        parameter; gone when the session ends.
- NEVER write the key value to any file.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on class name and
  Task<R> id. Each branch reads JSON from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    planner.json — three lists keyed by task id:
      "DRAFT_PLAN" → 4–6 PlanLedger entries (2–4 targetFiles, 4–8 steps).
      "DECIDE_ACTION" → 4–6 NextAction entries covering Continue across all
      three action kinds, one Replan, one Commit, one Fail.
      "COMPOSE_COMMIT" → 4–6 CommitSummary entries with a commit message,
      1–4 filesChanged, testsPassed=true, producedAt.
    reader.json — 6 ActionResult entries (actionKind=READ_FILE, ok=true,
      content fields are 10–20 line file excerpts from the fixture files).
    editor.json — 5 ActionResult entries (actionKind=EDIT_FILE, ok=true,
      content holds a unified-diff; one entry proposes a write outside
      /workspace/ to trigger the guardrail test).
    runner.json — 5 ActionResult entries (actionKind=RUN_TESTS, ok=true/
      false, testOutput holds Maven BUILD SUCCESS or pytest FAILED output).
- MockModelProvider.seedFor(sessionId) makes selection deterministic per
  session id.

Constraints — see AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent base class clause reads `extends AutonomousAgent`.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  proposeStep, dispatchStep, decideStep, commitStep.
- Lesson 6: Optional<T> for every nullable field on SessionRow and Session.
- Lesson 7: PlannerTasks.java and AgentTasks.java declaring every Task<R>.
- Lesson 8: conservative defaults claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: HTTP port 9802 in application.conf.
- Lesson 11: source.platform never appears in user-facing surfaces.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND
  themeVariables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. No hidden zombie panels.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue, and surface a summary at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
