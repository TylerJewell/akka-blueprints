# SPEC — todoist-organizer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Todoist AI Inbox Organizer.
**One-line pitch:** A background worker polls the Todoist inbox, classifies each task into a project and label set, and writes the result back — governed by a before-tool-call guardrail and a periodic eval sampler.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of a single AI primitive (`TaskClassifierAgent`). Specifically:

- A **before-tool-call guardrail** on the `updateTask` tool enforces "only well-classified tasks may mutate Todoist state — low-confidence classifications are blocked before any write occurs".
- An **eval-periodic** sampler running every 60 minutes scores a sample of updated tasks for classification correctness, catching model drift before it pollutes the user's workspace.

The result is a system where every Todoist write has passed an explicit confidence gate, and continuous quality measurement flags degradation automatically.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live organizer log: every fetched task, its classification, its guardrail verdict, and its final status.
2. `TodoistPoller` (TimedAction) ticks every 30 s and pulls uncategorized tasks from the in-process `TodoistSimulator`. (Drips canned tasks — replace with the real REST client for deployment.)
3. For each new task, an `OrganizerWorkflow` starts: `TaskClassifierAgent` classifies it, then the guardrail checks confidence before any write.
4. If the guardrail passes, `OrganizerWorkflow` calls `updateTask` and the task transitions to UPDATED.
5. If the guardrail blocks the write (confidence too low or ambiguous project), the task transitions to SKIPPED with a reason recorded.
6. `EvalRunner` (TimedAction) ticks every 60 minutes, picks up to 5 UPDATED tasks without an `evalScore`, calls a judge agent, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TodoistPoller` | `TimedAction` | Pulls uncategorized tasks from the simulator every 30 s. | scheduler | `TaskQueue` |
| `TaskQueue` | `EventSourcedEntity` | Append-only audit log of `TaskFetched` events. | `TodoistPoller` | `OrganizerWorkflow` (via Consumer) |
| `TaskFetchConsumer` | `Consumer` | Subscribes to `TaskFetched` events; starts one `OrganizerWorkflow` per task. | `TaskQueue` events | `OrganizerWorkflow`, `TodoistTaskEntity` |
| `TaskClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a task into a project, priority, and label set. Returns `ClassificationResult`. | invoked by Workflow | returns `ClassificationResult` |
| `OrganizerWorkflow` | `Workflow` | Per-task orchestration: classify → guardrail → update or skip. | `TaskFetchConsumer` | `TodoistTaskEntity` |
| `TodoistTaskEntity` | `EventSourcedEntity` | Lifecycle per task: fetched → classified → updated / skipped / failed. | `OrganizerWorkflow` | `OrganizerView` |
| `OrganizerView` | `View` | Read-model row per task for the UI. | `TodoistTaskEntity` events | `OrganizerEndpoint` |
| `EvalRunner` | `TimedAction` | Every 60 min, samples UPDATED tasks without scores; calls judge; writes `EvalScored`. | scheduler | `TodoistTaskEntity` |
| `OrganizerEndpoint` | `HttpEndpoint` | `/api/organizer/*` — list, get, SSE. | — | `OrganizerView`, `TodoistTaskEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record TodoistTask(String taskId, String content, String description,
                   Optional<String> projectId, List<String> labels,
                   String priority, Instant fetchedAt) {}

record ClassificationResult(
    String targetProjectId, String targetProjectName,
    List<String> targetLabels, String priorityLevel,
    String confidence, String reason) {}
// confidence: "high" | "medium" | "low"
// priorityLevel: "p1" | "p2" | "p3" | "p4"

record GuardrailVerdict(boolean allowed, String reason) {}

record UpdateRecord(String targetProjectId, String targetProjectName,
                    List<String> appliedLabels, String priorityLevel,
                    Instant updatedAt) {}

record EvalResult(int score, String rationale) {}

record OrganizerTask(
    String taskId,
    TodoistTask original,
    Optional<ClassificationResult> classification,
    Optional<GuardrailVerdict> guardrailVerdict,
    Optional<UpdateRecord> update,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    TaskStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus {
    FETCHED, CLASSIFIED, GUARDRAIL_BLOCKED, UPDATED, SKIPPED, FAILED
}
```

Events on `TodoistTaskEntity`: `TaskFetched`, `TaskClassified`, `GuardrailChecked`, `TaskUpdated`, `TaskSkipped`, `TaskFailed`, `EvalScored`.

Events on `TaskQueue`: `TaskFetched` (raw audit log entry).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/organizer` — list all tasks. Optional `?status=…`.
- `GET /api/organizer/{id}` — one task.
- `GET /api/organizer/sse` — Server-Sent Events for every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Todoist AI Inbox Organizer</title>`.

App UI tab shows the **live organizer log** with a column per status, classification chips, guardrail verdict chips, and eval score chips.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on the `updateTask` tool: blocks the write if `classification.confidence` is `"low"` or if `targetProjectId` is not in the configured project allow-list. Records `GuardrailVerdict` on the entity.
- **E1 — eval-periodic** (every 60 minutes): scores updated tasks on classification accuracy (correct project/labels vs. a sample of human-verified ground truth).

## 9. Agent prompts

- `TaskClassifierAgent` → `prompts/task-classifier.md`. Typed classifier. Returns one `ClassificationResult` per task.
- `EvalJudge` (used by `EvalRunner`) → `prompts/eval-judge.md`. Scores a completed classification on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a task; it appears in the UI within 30 s; passes classify → guardrail pass → UPDATED.
2. **J2** — A low-confidence task is blocked by the guardrail; status is SKIPPED; reason visible in the UI.
3. **J3** — Eval Runner scores at least one UPDATED task within 60 minutes; score visible in the UI.
4. **J4** — The audit log (`GET /api/organizer`) shows `guardrailVerdict` on every completed task.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named todoist-organizer demonstrating the continuous-monitor × ops-automation
cell. Runs out of the box (in-process Todoist simulator; no real Todoist API key required).
Maven group io.akka.samples. Artifact id continuous-monitor-ops-automation-todoist-organizer.
Java package io.akka.samples.todoistaiinboxorganizer. HTTP port 9108.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) TaskClassifierAgent — classifier. System prompt loaded from
  prompts/task-classifier.md. Input: TodoistTask{taskId, content, description,
  Optional<String> projectId, List<String> labels, String priority, Instant fetchedAt}.
  Output: ClassificationResult{targetProjectId, targetProjectName, targetLabels: List<String>,
  priorityLevel: "p1"|"p2"|"p3"|"p4", confidence: "high"|"medium"|"low", reason: String}.
  Defaults to low confidence when the task description is ambiguous.

- 1 Agent (typed, NOT autonomous) EvalJudge — scorer. System prompt from prompts/eval-judge.md.
  Input: TodoistTask + ClassificationResult + UpdateRecord. Output: EvalResult{score: Integer
  1–5, rationale: String}.

- 1 Workflow OrganizerWorkflow per task with steps: classifyStep -> guardrailStep -> conditional:
  guardrailVerdict.allowed == true -> updateStep -> end with TaskUpdated;
  guardrailVerdict.allowed == false -> end with TaskSkipped(reason).
  classifyStep wraps the TaskClassifierAgent call with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(15)). guardrailStep is a synchronous in-workflow check (no
  LLM call) that reads the ClassificationResult confidence and project allow-list from config;
  it emits GuardrailChecked event to the entity. updateStep calls the TodoistSimulator
  updateTask method and emits TaskUpdated.

- 2 EventSourcedEntities:
  * TaskQueue — append-only audit log of inbound tasks. Command fetch(TodoistTask) emits
    TaskFetched{task}.
  * TodoistTaskEntity (one per taskId) — full per-task lifecycle. State OrganizerTask{taskId,
    original: TodoistTask, Optional<ClassificationResult> classification,
    Optional<GuardrailVerdict> guardrailVerdict, Optional<UpdateRecord> update,
    Optional<Integer> evalScore, Optional<String> evalRationale, TaskStatus status,
    Instant createdAt, Optional<Instant> finishedAt}. TaskStatus enum: FETCHED, CLASSIFIED,
    GUARDRAIL_BLOCKED, UPDATED, SKIPPED, FAILED. Events: TaskFetched, TaskClassified,
    GuardrailChecked, TaskUpdated, TaskSkipped, TaskFailed, EvalScored. Commands: registerTask,
    attachClassification, recordGuardrailVerdict, markUpdated, markSkipped, markFailed,
    recordEval, getTask. emptyState() returns OrganizerTask.initial("") without
    commandContext() reference.

- 1 Consumer TaskFetchConsumer subscribed to TaskQueue events; for each TaskFetched, calls
  TodoistTaskEntity.registerTask with the raw task, then starts an OrganizerWorkflow with
  taskId as the workflow id.

- 1 View OrganizerView with row type OrganizerTaskRow (mirrors OrganizerTask).
  Table updater consumes TodoistTaskEntity events. ONE query getAllTasks
  SELECT * AS tasks FROM organizer_view. No WHERE status filter — caller filters client-side.

- 2 TimedActions:
  * TodoistPoller — every 30s, reads next entry from
    src/main/resources/sample-events/todoist-tasks.jsonl and calls TaskQueue.fetch.
  * EvalRunner — every 60 minutes, queries OrganizerView.getAllTasks, picks up to 5 UPDATED
    tasks without an evalScore (oldest-first), calls EvalJudge with a 1–5 rubric per task,
    then calls TodoistTaskEntity.recordEval(score, rationale) per task.

- 2 HttpEndpoints:
  * OrganizerEndpoint at /api with GET /organizer, GET /organizer/{id}, GET /organizer/sse,
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Approve/reject not needed — no HITL gate in this baseline.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- OrganizerTasks.java declaring two Task<R> constants: CLASSIFY (ClassificationResult),
  EVAL (EvalResult).
- Domain records TodoistTask, ClassificationResult, GuardrailVerdict, UpdateRecord, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9108 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/todoist-tasks.jsonl with 10 canned task lines covering
  UPDATED (high-confidence, clear project match), SKIPPED (low-confidence, ambiguous), and
  FAILED edge cases.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail before-tool-call
  scope-enforcement, E1 eval-periodic performance-monitor. Matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root with decisions.authority_level = automated-with-guard,
  oversight.human_in_loop = false, oversight.automated_guardrail = true,
  failure.failure_modes including "incorrect-project-assignment" and
  "low-confidence-task-silently-moved"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/task-classifier.md and prompts/eval-judge.md loaded as agent system prompts.
- README.md at project root matching the blueprint root README.md structure.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Browser title
  exactly: <title>Akka Sample: Todoist AI Inbox Organizer</title>. App UI tab uses a
  two-column layout (left = live task list with status pills, classification chips, guardrail
  verdict chips; right = selected task detail with classification breakdown and eval score).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    task-classifier.json — 10–14 ClassificationResult entries spanning
      UPDATED (high-confidence, clear project slug, 1–3 labels), SKIPPED
      (low-confidence, ambiguous project, reason "insufficient description"),
      and an edge case with confidence "medium" that the guardrail blocks
      if below the configured threshold. Covers "Work", "Personal",
      "Learning", "Admin" as target projects.
    eval-judge.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric (correct project / accurate labels /
      priority alignment).
- A `MockModelProvider.seedFor(taskId)` helper makes per-task selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- Agent (typed) never silently downgraded from the SPEC.
- The guardrail runs as a synchronous in-workflow check BEFORE the updateTask tool call —
  not inside the agent's prompt and not after the Todoist API has been called.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
