# SPEC — orchestrator-workers-pattern

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Orchestrator-Workers Workflow.
**One-line pitch:** Submit a multi-file code-edit task; an orchestrator decomposes it into per-file instructions, dispatches workers in parallel, and synthesises the final changeset with a guardrail on every tool call.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to multiple FileEditor AutonomousAgents in parallel, collects their edited-file outputs, and asks an Orchestrator AutonomousAgent to synthesise the final changeset. The blueprint also demonstrates a **before-tool-call guardrail** that gates every file-write action a worker agent attempts, and an **eval-event** governance control that samples the orchestrator's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a task description and a list of target files via the form.

1. The system creates an `EditTask` record in `PLANNING` and starts an `EditWorkflow`.
2. The Orchestrator decomposes the task into per-file edit instructions, one `FileInstruction` per target file.
3. The workflow forks: a `FileEditor` worker runs for each target file concurrently. Each `FileEditor` calls the `applyEdit` tool; before every tool call the before-tool-call guardrail checks that the target path is within the allowed set. If the guardrail rejects, the worker ends with `BLOCKED`; otherwise the edit proceeds.
4. An `EditReviewer` agent checks each edited file for correctness and consistency, returning a `ReviewVerdict`.
5. The Orchestrator merges all `EditedFile` and `ReviewVerdict` outputs into a `Changeset { files, summary, qualityVerdict }`.
6. If any worker returns `BLOCKED`, the task enters `BLOCKED`. If all workers complete and all reviews pass, the task enters `COMPLETED`. If a worker times out after 60 seconds, the workflow synthesises from whichever files returned and the task enters `DEGRADED`.

A `TaskSimulator` (TimedAction) drips a sample edit task every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `Orchestrator` | `AutonomousAgent` | Decomposes the edit task into per-file instructions; synthesises the final changeset from worker outputs. | `EditWorkflow` | returns typed result to workflow |
| `FileEditor` | `AutonomousAgent` | Applies targeted edits to one file. Calls `applyEdit` tool gated by the before-tool-call guardrail. | `EditWorkflow` | — |
| `EditReviewer` | `AutonomousAgent` | Reviews one edited file for correctness and consistency; returns a `ReviewVerdict`. | `EditWorkflow` | — |
| `EditWorkflow` | `Workflow` | Coordinates the parallel fan-out, review collection, and synthesis. | `EditEndpoint`, `EditJobConsumer` | `EditTaskEntity` |
| `EditTaskEntity` | `EventSourcedEntity` | Holds the task's lifecycle (planning → in-progress → completed / degraded / blocked). | `EditWorkflow` | `EditTaskView` |
| `EditJobQueue` | `EventSourcedEntity` | Logs each submitted task for replay and audit. | `EditEndpoint`, `TaskSimulator` | `EditJobConsumer` |
| `EditTaskView` | `View` | List-of-tasks read model. | `EditTaskEntity` events | `EditEndpoint` |
| `EditJobConsumer` | `Consumer` | Listens to `EditJobQueue` events and starts one `EditWorkflow` per submission. | `EditJobQueue` events | `EditWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample edit task every 60 s. | scheduler | `EditJobQueue` |
| `QualitySampler` | `TimedAction` | Samples one completed task every 5 minutes for eval scoring; emits a `QualityScored` event. | scheduler | `EditTaskEntity` |
| `EditEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `EditTaskView`, `EditJobQueue`, `EditTaskEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TaskRequest(String description, List<String> targetFiles, String requestedBy) {}

record FileInstruction(String filePath, String instruction) {}
record DecompositionPlan(List<FileInstruction> instructions, String objective) {}

record EditedFile(String filePath, String editedContent, String diffSummary) {}

record ReviewVerdict(String filePath, boolean approved, String feedback) {}

record Changeset(
    List<EditedFile> files,
    List<ReviewVerdict> reviews,
    String summary,
    String qualityVerdict,
    Instant completedAt
) {}

record EditTask(
    String taskId,
    String description,
    List<String> targetFiles,
    TaskStatus status,
    Optional<DecompositionPlan> plan,
    Optional<List<EditedFile>> editedFiles,
    Optional<List<ReviewVerdict>> reviews,
    Optional<Changeset> changeset,
    Optional<String> failureReason,
    Optional<Integer> qualityScore,
    Optional<String> qualityRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { PLANNING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED }
```

### Events (on `EditTaskEntity`)

`TaskCreated`, `PlanReady`, `FileEdited`, `FileReviewed`, `TaskCompleted`, `TaskDegraded`, `TaskBlocked`, `QualityScored`.

### Events (on `EditJobQueue`)

`TaskSubmitted { taskId, description, targetFiles, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ description, targetFiles }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=PLANNING|IN_PROGRESS|COMPLETED|DEGRADED|BLOCKED`.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Orchestrator-Workers Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task description and target files, live list of tasks with status pills, expand-row to see the decomposition plan, edited files, review verdicts, changeset summary, and quality score.

Browser title: `<title>Akka Sample: Orchestrator-Workers Workflow</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — before-tool-call guardrail** (`before-tool-call` on `FileEditor`): gates every `applyEdit` tool invocation; rejects writes to paths outside the declared `targetFiles` set and blocks write actions flagged as destructive (e.g., whole-file deletion). Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `QualitySampler` (TimedAction) picks one completed task every 5 minutes and emits a `QualityScored` event with a 1–5 score and a short rationale assessing changeset coherence.

## 9. Agent prompts

- `Orchestrator` → `prompts/orchestrator.md`. Decomposes the edit task into per-file instructions; later synthesises the final changeset.
- `FileEditor` → `prompts/file-editor.md`. Applies targeted edits to one file.
- `EditReviewer` → `prompts/edit-reviewer.md`. Reviews one edited file.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a task with two target files; task progresses PLANNING → IN_PROGRESS → COMPLETED within 90 s; UI reflects each transition via SSE.
2. **J2** — Worker attempts a write to a path outside the declared file set; the before-tool-call guardrail rejects it and the task enters BLOCKED.
3. **J3** — Inject a worker timeout (set `FileEditor` timeout to 1 s); task enters DEGRADED with whichever files completed.
4. **J4** — Wait after a successful completion; the task's row shows a quality score from `QualitySampler`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named orchestrator-workers-pattern demonstrating the
delegation-supervisor-workers × dev-code cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-dev-code-orchestrator-workers-pattern.
Java package io.akka.samples.orchestratorworkersworkflow. Akka 3.6.0.
HTTP port 9579.

Components to wire (exactly):
- 3 AutonomousAgents:
  * Orchestrator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/orchestrator.md. Returns DecompositionPlan{instructions, objective} for DECOMPOSE
    and Changeset{files, reviews, summary, qualityVerdict, completedAt} for SYNTHESISE.
  * FileEditor — capability(TaskAcceptance.of(EDIT).maxIterationsPerTask(4)). System prompt from
    prompts/file-editor.md. Returns EditedFile{filePath, editedContent, diffSummary}.
    Before every applyEdit tool call the before-tool-call guardrail checks that filePath is
    within the task's declared targetFiles; if not, the guardrail rejects and the step ends
    with TaskBlocked.
  * EditReviewer — capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System prompt
    from prompts/edit-reviewer.md. Returns ReviewVerdict{filePath, approved, feedback}.

- 1 Workflow EditWorkflow with steps:
  decomposeStep -> [parallel per file] editStep(i) -> [parallel] reviewStep(i) ->
  synthesiseStep -> emitStep.
  decomposeStep calls forAutonomousAgent(Orchestrator.class, DECOMPOSE).
  editStep(i) and reviewStep(i) run in parallel across files using CompletionStage allOf;
  each editStep governed by WorkflowSettings.builder().stepTimeout(60s); each reviewStep
  governed by stepTimeout(30s). On any editStep timeout, transition to degradeStep that
  synthesises from completed files and ends with TaskDegraded. synthesiseStep calls
  forAutonomousAgent(Orchestrator.class, SYNTHESISE) with all EditedFile and ReviewVerdict
  outputs; give synthesiseStep a 90s stepTimeout. WorkflowSettings is nested inside
  Workflow — no import.

- 1 EventSourcedEntity EditTaskEntity holding state EditTask{taskId, description,
  targetFiles: List<String>, TaskStatus, Optional<DecompositionPlan> plan,
  Optional<List<EditedFile>> editedFiles, Optional<List<ReviewVerdict>> reviews,
  Optional<Changeset> changeset, Optional<String> failureReason,
  Optional<Integer> qualityScore, Optional<String> qualityRationale,
  Instant createdAt, Optional<Instant> finishedAt}.
  TaskStatus enum: PLANNING, IN_PROGRESS, COMPLETED, DEGRADED, BLOCKED.
  Events: TaskCreated, PlanReady, FileEdited, FileReviewed, TaskCompleted,
  TaskDegraded, TaskBlocked, QualityScored.
  Commands: createTask, recordPlan, recordFileEdit, recordReview, complete,
  degrade, block, recordQuality, getTask. emptyState() returns EditTask.initial
  with no commandContext() reference.

- 1 EventSourcedEntity EditJobQueue with command submitTask(description,
  targetFiles, requestedBy) emitting TaskSubmitted{taskId, description,
  targetFiles, requestedBy, submittedAt}.

- 1 View EditTaskView with row type EditTaskRow (mirrors EditTask minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes
  EditTaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM edit_task_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters
  client-side.

- 1 Consumer EditJobConsumer subscribed to EditJobQueue events; on TaskSubmitted
  starts an EditWorkflow with the taskId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/edit-tasks.jsonl and calls
    EditJobQueue.submitTask.
  * QualitySampler — every 5 minutes, queries EditTaskView.getAllTasks, picks the
    oldest COMPLETED task without a qualityScore, runs a 1–5 rubric judge over the
    changeset content, then calls EditTaskEntity.recordQuality(score, rationale).

- 2 HttpEndpoints:
  * EditEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- EditTasks.java declaring four Task<R> constants: DECOMPOSE (DecompositionPlan),
  EDIT (EditedFile), REVIEW (ReviewVerdict), SYNTHESISE (Changeset).
- Domain records TaskRequest, FileInstruction, DecompositionPlan, EditedFile,
  ReviewVerdict, Changeset.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9579 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/edit-tasks.jsonl with 8 canned edit-task lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 before-tool-call guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  code-editing-automation, decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/orchestrator.md, prompts/file-editor.md, prompts/edit-reviewer.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Orchestrator-Workers Workflow",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid
  diagrams + click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html; unanswered .qb opacity 0.45),
  Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows), App UI (form + live list with status pills). Browser title
  exactly: <title>Akka Sample: Orchestrator-Workers Workflow</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (orchestrator.json,
  file-editor.json, edit-reviewer.json), picks one entry pseudo-randomly per
  call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    orchestrator.json — list of either DecompositionPlan or Changeset objects.
      4–6 DecompositionPlan entries (each with 2–4 FileInstruction items and
      an objective string) and 4–6 Changeset entries (each with 2–4 EditedFile
      items, 2–4 ReviewVerdict items, an 80–120 word summary, qualityVerdict
      = "ok").
    file-editor.json — 4–6 EditedFile entries, each with a realistic filePath,
      a short editedContent snippet, and a one-line diffSummary describing
      what changed.
    edit-reviewer.json — 4–6 ReviewVerdict entries, each with approved = true
      and a brief feedback string confirming the edit is consistent with
      surrounding code.
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic
  per task id so the same task produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout
  (60s editStep, 30s reviewStep, 90s synthesiseStep); default 5s is too short
  for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion EditTasks.java declaring
  Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9579 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  theme variables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state names
  render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  NodeList index; delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf across all file workers,
  NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
