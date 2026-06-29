# SPEC — akka-memory-first-coding-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Code Memory-First Coding Agent.
**One-line pitch:** Run `/init` to have the agent deep-research your codebase, write persistent memory blocks about it, and rewrite its own system prompt — then ask for code edits backed by that memory, with every file-write and code-execution step gated by a guardrail and a test gate.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with a two-phase design: a **research phase** (triggered by `/init`) that builds project memory, and an **edit phase** (triggered by `/edit`) that uses that memory to plan and apply code changes.

In the research phase, the `ResearchPlannerAgent` scans the codebase index and produces a `ResearchPlan`. A loop of `CodeReaderAgent` calls fills in `FileInsight` records. `MemoryWriterAgent` synthesises those insights into named `MemoryBlock` entries stored on `ProjectEntity` and rewrites the agent's own system prompt from the blocks.

In the edit phase, `EditPlannerAgent` reads the memory blocks and the edit request to produce a `PatchPlan`. A loop of `EditExecutorAgent` calls applies `FileEdit` operations one at a time, each gated by:

- a **before-tool-call guardrail** that checks path scope and operation type,
- a **ci-gate** (test gate) that runs the fixture test suite after each patch and blocks promotion on failure,
- an **operator halt** path for destructive operations.

The blueprint also demonstrates how an agent rewrites its own system prompt at runtime — `MemoryWriterAgent` calls a `ProjectEntity` command that persists the new prompt text, and `ResearchWorkflow` reloads that prompt into subsequent agent invocations.

## 3. User-facing flows

The user opens the App UI tab.

**Init flow:**
1. The user clicks **Init project** (or the simulator triggers it at startup). The system creates a `Project` record in `RESEARCHING` and starts a `ResearchWorkflow`.
2. `ResearchPlannerAgent` scans the fixture index and returns a `ResearchPlan { filesToRead, questions }`.
3. A loop of `CodeReaderAgent` calls produces `FileInsight { fileName, language, summary, keySymbols }` records.
4. `MemoryWriterAgent` synthesises the insights into a list of `MemoryBlock { name, content, sourceFiles }` and a rewritten `systemPrompt` string.
5. `ProjectEntity` stores the blocks and prompt. `ResearchWorkflow` completes with status `READY`.

**Edit flow:**
1. The user types an edit request ("Add input validation to the auth handler") and clicks **Submit edit**.
2. The system creates an `EditSession` record in `PLANNING` and starts an `EditWorkflow`.
3. `EditPlannerAgent` reads the memory blocks and returns a `PatchPlan { edits: List<FileEdit> }`.
4. The workflow enters the apply loop. Each iteration:
   - The **before-tool-call guardrail** vets the `FileEdit` against path scope and allowed operation types. On rejection the workflow records a `PatchBlocked` entry and asks the planner to revise.
   - `EditExecutorAgent` applies the edit and returns a `PatchResult { ok, diff, errorReason }`.
   - The **test gate** runs the fixture test suite (`FixtureTestRunner.run`). On failure the workflow records `TestsFailed` and ends the session in `TESTS_FAILED`.
   - On success the workflow records `PatchApplied` and advances to the next edit.
5. When all edits are applied and tests pass, the session moves to `COMPLETED`.
6. The operator can press **Halt destructive edits** in the dashboard. The in-flight apply finishes; the session ends in `HALTED`.

A `ResearchSimulator` (TimedAction) triggers one sample project init on startup.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchPlannerAgent` | `AutonomousAgent` | Scans codebase index; produces `ResearchPlan`. | `ResearchWorkflow` | returns `ResearchPlan` to workflow |
| `CodeReaderAgent` | `AutonomousAgent` | Reads fixture file excerpt; answers question; returns `FileInsight`. | `ResearchWorkflow` | — |
| `MemoryWriterAgent` | `AutonomousAgent` | Synthesises `FileInsight` records into `MemoryBlock` list and new system prompt. | `ResearchWorkflow` | — |
| `EditPlannerAgent` | `AutonomousAgent` | Reads memory blocks and edit request; returns `PatchPlan`. | `EditWorkflow` | — |
| `EditExecutorAgent` | `AutonomousAgent` | Applies one `FileEdit`; returns `PatchResult`. | `EditWorkflow` | — |
| `ResearchWorkflow` | `Workflow` | Drives init path: scan → read-loop → synthesise → persist-memory. | `ProjectEndpoint`, `ResearchSimulator` | `ProjectEntity` |
| `EditWorkflow` | `Workflow` | Drives edit path: plan → [loop] guardrail → apply → test-gate → record → decide. | `SessionRequestConsumer` | `EditSessionEntity`, `ProjectEntity` |
| `ProjectEntity` | `EventSourcedEntity` | Holds memory blocks, current system prompt, and project lifecycle. | `ResearchWorkflow` | `ProjectView` |
| `EditSessionEntity` | `EventSourcedEntity` | Holds one session's patch plan, applied edits, test results, and halt state. | `EditWorkflow` | `ProjectView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by `"global"`. | `ProjectEndpoint` | `EditWorkflow` (polls) |
| `ProjectView` | `View` | Read model for projects and sessions. | `ProjectEntity` + `EditSessionEntity` events | `ProjectEndpoint` |
| `SessionRequestConsumer` | `Consumer` | Subscribes to `ProjectEntity` events; starts an `EditWorkflow` per `EditSessionRequested` event. | `ProjectEntity` events | `EditWorkflow` |
| `ResearchSimulator` | `TimedAction` | On startup fires once; reads first project from `sample-events/projects.jsonl` and starts a `ResearchWorkflow`. | scheduler | `ProjectEntity` |
| `StaleSessionMonitor` | `TimedAction` | Every 30 s, marks any session stuck in `APPLYING` past 5 minutes as `TIMED_OUT`. | scheduler | `EditSessionEntity` |
| `ProjectEndpoint` | `HttpEndpoint` | `/api/projects/*`, `/api/sessions/*` — init, list, get, edit, halt, SSE. | — | `ProjectView`, `ProjectEntity`, `EditSessionEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record InitRequest(String projectPath, String requestedBy) {}

record ResearchPlan(
    List<String> filesToRead,
    List<String> questions
) {}

record FileInsight(
    String fileName,
    String language,
    String summary,
    List<String> keySymbols
) {}

record MemoryBlock(
    String name,
    String content,
    List<String> sourceFiles
) {}

record ProjectMemory(
    List<MemoryBlock> blocks,
    String systemPrompt,
    Instant builtAt
) {}

record EditRequest(String projectId, String instruction, String requestedBy) {}

record FileEdit(
    String filePath,
    EditKind kind,
    String patch
) {}

record PatchPlan(
    List<FileEdit> edits,
    String rationale
) {}

record PatchResult(
    String filePath,
    boolean ok,
    Optional<String> diff,
    Optional<String> errorReason
) {}

record TestResult(
    boolean passed,
    int total,
    int failed,
    Optional<String> output
) {}

record PatchEntry(
    int attempt,
    FileEdit edit,
    PatchVerdict verdict,
    Optional<PatchResult> result,
    Optional<TestResult> testResult,
    Optional<String> blocker,
    Instant recordedAt
) {}

record Project(
    String projectId,
    String projectPath,
    ProjectStatus status,
    Optional<ResearchPlan> plan,
    Optional<ProjectMemory> memory,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> readyAt
) {}

record EditSession(
    String sessionId,
    String projectId,
    String instruction,
    SessionStatus status,
    Optional<PatchPlan> patchPlan,
    List<PatchEntry> patchLog,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EditKind { INSERT, REPLACE, DELETE }
enum PatchVerdict { APPLIED, BLOCKED_BY_GUARDRAIL, FAILED, TESTS_FAILED }
enum ProjectStatus { RESEARCHING, READY, FAILED }
enum SessionStatus { PLANNING, APPLYING, COMPLETED, TESTS_FAILED, FAILED, HALTED, TIMED_OUT }
```

### Events (`ProjectEntity`)

`ProjectCreated`, `ResearchPlanReady`, `MemoryBuilt`, `SystemPromptRewritten`, `ProjectFailed`, `EditSessionRequested`.

### Events (`EditSessionEntity`)

`SessionCreated`, `PatchPlanReady`, `PatchBlocked`, `PatchApplied`, `TestsPassed`, `TestsFailed`, `SessionCompleted`, `SessionFailed`, `SessionHalted`, `SessionTimedOut`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/projects/init` — body `{ projectPath, requestedBy? }` → `202 { projectId }`. Starts a ResearchWorkflow.
- `GET /api/projects` — list all projects with status and memory-block count.
- `GET /api/projects/{id}` — one project (full memory blocks + system prompt).
- `GET /api/projects/{id}/memory` — the current `ProjectMemory` (blocks + prompt).
- `POST /api/projects/{id}/edit` — body `{ instruction, requestedBy? }` → `202 { sessionId }`. Starts an EditWorkflow.
- `GET /api/sessions` — list all edit sessions. Optional `?projectId=...` filter.
- `GET /api/sessions/{id}` — one session (patch plan + patch log).
- `GET /api/sessions/sse` — server-sent events stream of session and project changes.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Code Memory-First Coding Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — project init panel, edit request form, operator halt/resume control, live project list with memory-block count, live session list with patch log, expand-row to see the patch plan, patch log entries, and test results.

Browser title: `<title>Akka Sample: Code Memory-First Coding Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `EditExecutorAgent`): every `FileEdit` is checked against (a) an allowed-path list (`/workspace/` only), (b) an operation-type policy that forbids `DELETE` on files outside a designated deletable set, and (c) a pattern deny-list for edits touching shell-script files with world-writable mode bits. Blocking. Rejection → `PatchBlocked` entry + revised plan request.
- **CI1 — test gate** (`ci-gate`, flavor `test-gate`): after every `PatchApplied` event, `FixtureTestRunner.run` executes the project's fixture test suite. Non-zero exit → `TestsFailed` event; the session ends in `TESTS_FAILED` and the patch is not promoted to project memory. Passing tests → `TestsPassed` event.
- **HO1 — operator halt** (`halt`, flavor `operator-regulator-stop`): the operator dashboard exposes a **Halt destructive edits** button backed by `POST /api/control/halt`. `EditWorkflow` polls `SystemControlEntity` before each apply step. On `halted=true` it finishes the in-flight apply, then emits `SessionHalted` and ends in `HALTED`.

## 9. Agent prompts

- `ResearchPlannerAgent` → `prompts/research-planner.md`. Produces a `ResearchPlan`.
- `CodeReaderAgent` → `prompts/code-reader.md`. Reads a file excerpt and answers a question.
- `MemoryWriterAgent` → `prompts/memory-writer.md`. Synthesises insights into memory blocks and rewrites the system prompt.
- `EditPlannerAgent` → `prompts/edit-planner.md`. Produces a `PatchPlan` from memory blocks and the edit instruction.
- `EditExecutorAgent` → `prompts/edit-executor.md`. Applies one `FileEdit` and returns a `PatchResult`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Run `/init` on the seeded project. Status progresses `RESEARCHING → READY` within ~3 minutes. The UI's memory panel shows 4–8 `MemoryBlock` entries with non-empty content, and the system prompt field shows the rewritten prompt.
2. **J2** — Submit an edit request whose plan includes a `DELETE` on an out-of-scope path. The guardrail blocks the patch; the planner revises; the session either completes via a revised plan or ends in `FAILED` with a clear reason. The guarded operation never executes.
3. **J3** — Submit an edit request; the fixture test suite returns a failure on the resulting patch. The session ends in `TESTS_FAILED`. The memory blocks are not updated. The UI shows the test output in the expanded session row.
4. **J4** — Submit an edit request and click **Halt** while the session is `APPLYING`. The in-flight apply completes; no further edits run; the session ends in `HALTED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-memory-first-coding-agent demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact
planner-executor-dev-code-akka-memory-first-coding-agent. Java package
io.akka.samples.codememoryfirstcodingagent. Akka 3.6.0. HTTP port 9365.

Components to wire (exactly):
- 5 AutonomousAgents:
  * ResearchPlannerAgent — capability(TaskAcceptance.of(SCAN_CODEBASE)
      .maxIterationsPerTask(2)). System prompt from
      prompts/research-planner.md. Returns ResearchPlan.
  * CodeReaderAgent — capability(TaskAcceptance.of(READ_FILE)
      .maxIterationsPerTask(2)). Prompt from prompts/code-reader.md.
      Returns FileInsight.
  * MemoryWriterAgent — capability(TaskAcceptance.of(SYNTHESISE_MEMORY)
      .maxIterationsPerTask(3)). Prompt from prompts/memory-writer.md.
      Returns ProjectMemory (blocks + systemPrompt). This agent's effective
      system prompt must be re-loaded from ProjectEntity.memory.systemPrompt
      (if non-empty) at the start of every SYNTHESISE_MEMORY task so that
      self-rewrite takes effect on subsequent runs.
  * EditPlannerAgent — capability(TaskAcceptance.of(PLAN_EDITS)
      .maxIterationsPerTask(3)). Prompt from prompts/edit-planner.md.
      Returns PatchPlan.
  * EditExecutorAgent — capability(TaskAcceptance.of(APPLY_EDIT)
      .maxIterationsPerTask(2)). Prompt from prompts/edit-executor.md.
      Returns PatchResult.

- 2 Workflows:
  * ResearchWorkflow with steps:
    scanStep -> readFileStep (loop over filesToRead) -> synthesiseStep ->
    persistMemoryStep -> doneStep.
    Step timeouts (override settings() per Lesson 4):
      scanStep ofSeconds(45), readFileStep ofSeconds(60) (per iteration),
      synthesiseStep ofSeconds(90), persistMemoryStep ofSeconds(30).
    defaultStepRecovery(maxRetries(2).failoverTo(ResearchWorkflow::error)).
    scanStep calls ResearchPlannerAgent SCAN_CODEBASE.
    readFileStep loops: for each file in plan.filesToRead calls
    CodeReaderAgent READ_FILE; collects FileInsight list.
    synthesiseStep calls MemoryWriterAgent SYNTHESISE_MEMORY with the
    FileInsight list.
    persistMemoryStep calls ProjectEntity.buildMemory(memory) and
    ProjectEntity.rewritePrompt(memory.systemPrompt).

  * EditWorkflow with steps:
    planStep -> [loop entry] checkHaltStep -> guardrailStep -> applyStep ->
    testGateStep -> recordStep -> decideStep ->
    [back to checkHaltStep or to completeStep / failStep / haltedStep].
    Step timeouts (Lesson 4):
      planStep ofSeconds(60), applyStep ofSeconds(90),
      testGateStep ofSeconds(120), decideStep ofSeconds(30).
    defaultStepRecovery(maxRetries(2).failoverTo(EditWorkflow::error)).
    checkHaltStep reads SystemControlEntity.get; on halted=true transitions
    to haltedStep (emits SessionHalted).
    guardrailStep runs EditGuardrail.vet(FileEdit); on reject records
    PatchBlocked via EditSessionEntity.recordBlock and revises plan.
    applyStep calls EditExecutorAgent APPLY_EDIT for the current FileEdit.
    testGateStep calls FixtureTestRunner.run(projectId); on failure
    transitions to testFailedStep (emits TestsFailed, ends TESTS_FAILED).
    recordStep calls EditSessionEntity.recordPatch(entry).
    decideStep checks if more edits remain; if yes loops; if no transitions
    to completeStep.

- 2 EventSourcedEntities:
  * ProjectEntity holding Project state. Commands: createProject,
    recordPlan, buildMemory, rewritePrompt, failProject, requestEdit, getProject.
    Events: ProjectCreated, ResearchPlanReady, MemoryBuilt,
    SystemPromptRewritten, ProjectFailed, EditSessionRequested.
  * EditSessionEntity holding EditSession state. Commands: createSession,
    recordPatchPlan, recordBlock, recordPatch, recordTestPass, recordTestFail,
    completeSession, failSession, haltSession, timeoutSession, getSession.
    Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason,
  Optional<Instant> haltedAt}. Commands: requestHalt(reason), clearHalt, get.
  Events: HaltRequested, HaltCleared.

- 1 View ProjectView with two row types:
    ProjectRow (mirror of Project minus full memory payload — include only
    block count and first 200 chars of systemPrompt).
    SessionRow (mirror of EditSession minus full patchLog — include only
    last 3 PatchEntry records plus totalPatches count).
  TWO queries: getAllProjects SELECT * AS projects FROM project_view,
  getAllSessions SELECT * AS sessions FROM session_view.
  No WHERE filters — caller filters client-side (Lesson 2).

- 1 Consumer SessionRequestConsumer subscribed to ProjectEntity events;
  on EditSessionRequested starts an EditWorkflow with the sessionId and
  projectId from the event payload.

- 2 TimedActions:
  * ResearchSimulator — on startup fires once after 5 s delay; reads first
    line from src/main/resources/sample-events/projects.jsonl and calls
    ProjectEntity.createProject then starts a ResearchWorkflow.
  * StaleSessionMonitor — every 30 s, queries ProjectView.getAllSessions,
    filters APPLYING sessions whose createdAt is older than 5 minutes,
    calls EditSessionEntity.timeoutSession.

- 2 HttpEndpoints:
  * ProjectEndpoint at /api with POST /projects/init, GET /projects,
    GET /projects/{id}, GET /projects/{id}/memory, POST /projects/{id}/edit,
    GET /sessions, GET /sessions/{id}, GET /sessions/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ResearchTasks.java declaring three Task<R> constants: SCAN_CODEBASE
  (resultConformsTo ResearchPlan), READ_FILE (FileInsight),
  SYNTHESISE_MEMORY (ProjectMemory).
- EditTasks.java declaring two Task<R> constants: PLAN_EDITS (PatchPlan),
  APPLY_EDIT (PatchResult).
- Domain records as listed in SPEC §5.
- application/EditGuardrail.java — deterministic vetter. Reject if filePath
  is outside /workspace/, if kind == DELETE and filePath does not match
  allowedDeletePattern (src/test/**, build/**, target/**), if patch content
  contains a shebang line with world-writable chmod (chmod [ao]+w).
- application/FixtureTestRunner.java — deterministic simulator. Reads from
  src/main/resources/sample-data/test-results/<projectId>.json; returns a
  TestResult. If no file found, returns passed=true, total=0, failed=0.
  Includes one entry that returns passed=false with 2 failures for the J3
  acceptance test.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9365 and agent model-provider blocks
  for anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/projects.jsonl with 4 canned project
  definitions (projectPath, projectDescription, language).
- src/main/resources/sample-data/codebase/* — 8 short source files used
  by CodeReaderAgent (mix of Java, config, and markdown files forming a
  small believable microservice).
- src/main/resources/sample-data/test-results/*.json — per-project test
  result fixtures. Include one file returning failed=2 for J3.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint).
- eval-matrix.yaml at the project root with 3 controls (G1, CI1, HO1)
  and a matching simplified_view list.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls.
- prompts/research-planner.md, prompts/code-reader.md,
  prompts/memory-writer.md, prompts/edit-planner.md,
  prompts/edit-executor.md loaded at agent startup as system prompts.
- README.md at project root: title "Akka Sample: Code Memory-First
  Coding Agent", one-line pitch, prerequisites (integration form host
  software: None), generate-the-system, what-you-get, customise-before-
  generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Five tabs
  matching the formal exemplar: Overview, Architecture (4 mermaid
  diagrams + click-to-expand component table with syntax-highlighted Java
  snippets), Risk Survey (7 sub-tabs with risk-survey.yaml answers),
  Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows), App UI (project init panel + edit form +
  operator halt/resume + live project list with memory block count + live
  session list with patch log). Browser title exactly:
  <title>Akka Sample: Code Memory-First Coding Agent</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider shapes (option a):
- research-planner.json — 4 ResearchPlan entries (filesToRead 4–6 paths,
  questions 3–5 strings).
- code-reader.json — 8 FileInsight entries covering Java, YAML, and Markdown.
- memory-writer.json — 4 ProjectMemory entries; each has 4–6 MemoryBlock
  entries and a rewritten systemPrompt (2–3 sentences referencing the
  project's tech stack and naming conventions).
- edit-planner.json — 5 PatchPlan entries; each has 2–3 FileEdit operations
  spanning INSERT and REPLACE kinds.
- edit-executor.json — 6 PatchResult entries; ok=true with diff-shaped content.
  One entry's patch must include a path outside /workspace/ to trigger the
  guardrail in mock runs. One entry returns ok=false with errorReason for the
  FAILED verdict path.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent.
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on entity state records.
- Lesson 7: Companion ResearchTasks.java and EditTasks.java declaring every
  Task<R> constant.
- Lesson 8: model-name values verified against provider lineup.
  Conservative defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build", never "mvn akka:run".
- Lesson 10: HTTP port 9365.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the
  box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing file.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow in Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
