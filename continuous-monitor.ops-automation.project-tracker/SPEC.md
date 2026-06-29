# SPEC — project-tracker

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Project Manager Agent (Planner).
**One-line pitch:** A background worker watches a project board, assigns tasks to the right owners, and chases overdue items — with a before-tool-call guardrail protecting every action that touches a human.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with one governance mechanism layered on top of two AI primitives (`AssignmentAgent` and `NudgeAgent`). Specifically:

- A **before-tool-call guardrail** sits in front of every outbound action — `assignTask`, `sendNudge`, `escalateTask` — and checks that the action meets policy rules (owner capacity, assignment authority, message content policy) before the call is dispatched. Because task assignments and nudge messages affect humans directly, this is the primary governance check.

The result is a system where no automated action reaches a team member without passing a policy gate, and every dispatched action is recorded with its rationale so the audit trail is complete.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live board: every task, its status, its assignee (if any), due date, and (if overdue) the last nudge sent.
2. `PlannerPoller` (TimedAction) ticks every 20 s and drips simulated tasks into `BoardEventQueue`. (A simulator-style feed — reads from `sample-events/board-events.jsonl`.)
3. For each new task: `BoardSyncConsumer` (Consumer) normalises the event and emits `TaskCreated` on `PlannerTaskEntity`.
4. `PlannerWorkflow` starts: `AssignmentAgent` recommends an owner → guardrail checks the assignment → `assignTask` tool is called → entity transitions to ASSIGNED.
5. `StaleTaskChecker` (TimedAction) ticks every 5 minutes, finds tasks past their due date, and for each starts a nudge sub-flow: `NudgeAgent` drafts a message → guardrail checks content → `sendNudge` tool is called → entity records `NudgeSent`.
6. If a task receives three nudges without advancing, `StaleTaskChecker` escalates it: guardrail checks authority → `escalateTask` tool → entity transitions to ESCALATED.
7. `EvalRunner` (TimedAction) ticks every 60 minutes, picks up to 5 COMPLETED tasks without an `evalScore`, calls `EvalJudge`, and writes a score back via an `AssignmentEvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerPoller` | `TimedAction` | Drips simulated board events into `BoardEventQueue` every 20 s. | scheduler | `BoardEventQueue` |
| `BoardEventQueue` | `EventSourcedEntity` | Append-only log of raw `BoardEventReceived` events. | `PlannerPoller`, `BoardEndpoint` | `BoardSyncConsumer` |
| `BoardSyncConsumer` | `Consumer` | Reads `BoardEventReceived` events, normalises to typed domain records, emits `TaskCreated` / `TaskUpdated` on `PlannerTaskEntity`. | `BoardEventQueue` events | `PlannerTaskEntity`, `PlannerWorkflow` |
| `AssignmentAgent` | `Agent` (typed, NOT autonomous) | Recommends one team member for an unassigned task. | invoked by `PlannerWorkflow` | returns `AssignmentRecommendation` |
| `NudgeAgent` | `Agent` (typed, NOT autonomous) | Drafts a plain-text nudge for an overdue task. | invoked by `StaleTaskChecker` | returns `NudgeDraft` |
| `EvalJudge` | `Agent` (typed, NOT autonomous) | Scores a completed task's assignment on a 1–5 rubric. | invoked by `EvalRunner` | returns `EvalResult` |
| `PlannerWorkflow` | `Workflow` | Per-task orchestration: recommend owner → guardrail check → assign → wait for completion. | `BoardSyncConsumer` (one workflow per `TaskCreated`) | `PlannerTaskEntity` |
| `PlannerTaskEntity` | `EventSourcedEntity` | Lifecycle per task: created → assigned → in-progress → completed / overdue / escalated. | `PlannerWorkflow`, `StaleTaskChecker`, `BoardEndpoint` | `BoardView` |
| `BoardView` | `View` | Read-model row per task for the UI. | `PlannerTaskEntity` events | `BoardEndpoint` |
| `StaleTaskChecker` | `TimedAction` | Every 5 min: finds overdue tasks, triggers nudge sub-flow; escalates tasks with ≥ 3 nudges. | scheduler | `PlannerTaskEntity`, `NudgeAgent` |
| `EvalRunner` | `TimedAction` | Every 60 min: samples COMPLETED tasks without `evalScore`, calls `EvalJudge`, writes `AssignmentEvalScored`. | scheduler | `PlannerTaskEntity` |
| `BoardEndpoint` | `HttpEndpoint` | `/api/board/*` — list, get, complete, escalate, SSE. | — | `BoardView`, `PlannerTaskEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record RawBoardEvent(String eventId, String eventType, String taskId, String title,
                     String description, Optional<String> assigneeId, Optional<String> assigneeName,
                     Optional<Instant> dueDate, Instant occurredAt) {}

record TaskSummary(String taskId, String title, String description,
                   Optional<Instant> dueDate, TaskPriority priority) {}
enum TaskPriority { URGENT, HIGH, NORMAL, LOW }

record AssignmentRecommendation(String recommendedOwnerId, String recommendedOwnerName,
                                 String rationale, String confidence) {}

record NudgeDraft(String taskId, String ownerId, String message, NudgeTone tone, Instant draftedAt) {}
enum NudgeTone { FRIENDLY, DIRECT, ESCALATORY }

record DispatchResult(String actionId, String toolName, boolean dispatched,
                      Optional<String> blockReason, Instant attemptedAt) {}

record EvalResult(Integer score, String rationale) {}

record PlannerTask(
    String taskId,
    TaskSummary summary,
    Optional<AssignmentRecommendation> assignment,
    Optional<String> currentOwnerId,
    Optional<String> currentOwnerName,
    int nudgeCount,
    Optional<Instant> lastNudgedAt,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    TaskStatus status,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum TaskStatus {
    CREATED, ASSIGNED, IN_PROGRESS, OVERDUE, NUDGED, ESCALATED, COMPLETED, CANCELLED
}
```

Events on `PlannerTaskEntity`: `TaskCreated`, `TaskAssigned`, `TaskProgressStarted`, `TaskOverdue`, `NudgeSent`, `TaskEscalated`, `TaskCompleted`, `TaskCancelled`, `AssignmentEvalScored`.

Events on `BoardEventQueue`: `BoardEventReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/board` — list all tasks. Optional `?status=…`.
- `GET /api/board/{id}` — one task.
- `POST /api/board/{id}/complete` — body `{ resolvedBy }` → transitions to COMPLETED.
- `POST /api/board/{id}/escalate` — body `{ escalatedBy, reason }` → transitions to ESCALATED.
- `GET /api/board/sse` — Server-Sent Events for every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Project Manager Agent (Planner)</title>`.

App UI tab is the most distinctive: it shows the **live board** with a list of tasks sorted by status (OVERDUE first, then CREATED/ASSIGNED, then COMPLETED/ESCALATED). Each task card shows a status pill, priority badge, assignee chip, due date, and nudge count. The detail pane shows the full task body, assignment rationale, nudge history, and eval score when available.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on `assignTask`, `sendNudge`, and `escalateTask` tools: checks each outbound action against policy rules before the tool call executes. For assignments: verifies owner capacity and assignment authority. For nudges: checks message content policy and nudge-count ceiling. For escalations: verifies escalation authority. Blocks with a structured error if any check fails.

## 9. Agent prompts

- `AssignmentAgent` → `prompts/assignment-agent.md`. Typed classifier. Always returns one `AssignmentRecommendation`.
- `NudgeAgent` → `prompts/nudge-agent.md`. Drafts a nudge message for an overdue task owner.
- `EvalJudge` → `prompts/eval-judge.md`. Scores a completed task's assignment quality on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a new task; it appears in the UI within 20 s; transitions CREATED → ASSIGNED with a rationale visible.
2. **J2** — A task's simulated due date passes; `StaleTaskChecker` drafts and dispatches a nudge; entity records `NudgeSent`; nudge count increments.
3. **J3** — A task accumulates three nudges without advancing; entity transitions to ESCALATED.
4. **J4** — Mark a task as complete; within 60 minutes `EvalRunner` scores the assignment; score and rationale appear in the UI.
5. **J5** — Guardrail blocks an assignment when the recommended owner has no capacity; entity stays in CREATED; blockReason is logged.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named project-tracker demonstrating the continuous-monitor × ops-automation
cell. Runs out of the box (in-memory board simulator; no real Microsoft Graph integration).
Maven group io.akka.samples, artifact continuous-monitor-ops-automation-project-tracker.
Java package io.akka.samples.projectmanageragentplanner. HTTP port 9822.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) AssignmentAgent — recommends one owner per task. System
  prompt loaded from prompts/assignment-agent.md. Input: TaskSummary{taskId, title,
  description, dueDate: Optional<Instant>, priority: TaskPriority}. Output:
  AssignmentRecommendation{recommendedOwnerId, recommendedOwnerName, rationale, confidence:
  "high"|"medium"|"low"}. Defaults to the team lead under uncertainty.

- 1 Agent (typed, NOT autonomous) NudgeAgent — drafts a nudge message. System prompt from
  prompts/nudge-agent.md. Input: PlannerTask (current state). Output: NudgeDraft{taskId,
  ownerId, message, tone: NudgeTone (enum FRIENDLY/DIRECT/ESCALATORY), draftedAt}.
  Tone escalates with nudge count: nudgeCount 1 → FRIENDLY, 2 → DIRECT, ≥3 → ESCALATORY.

- 1 Agent (typed, NOT autonomous) EvalJudge — scores assignment quality. System prompt from
  prompts/eval-judge.md. Input: TaskSummary + AssignmentRecommendation + outcome (completed
  on time: boolean). Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow PlannerWorkflow per task with steps: recommendStep -> guardrailCheckStep ->
  assignStep -> monitorStep. recommendStep calls AssignmentAgent with
  WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(15)). guardrailCheckStep calls
  GuardrailChecker.checkAssignment(recommendation, taskSummary); if blocked, ends with
  TaskAssignmentBlocked event. assignStep calls assignTask tool (no-op stub); emits
  TaskAssigned. monitorStep polls PlannerTaskEntity.getTask every 30s; advances when status
  is COMPLETED or ESCALATED or CANCELLED.

- 2 EventSourcedEntities:
  * BoardEventQueue — append-only audit log. Command receiveEvent(RawBoardEvent) emits
    BoardEventReceived{rawBoardEvent}.
  * PlannerTaskEntity (one per taskId) — full per-task lifecycle. State
    PlannerTask{taskId, summary: TaskSummary{taskId, title, description,
    dueDate: Optional<Instant>, priority: TaskPriority}, assignment:
    Optional<AssignmentRecommendation>, currentOwnerId: Optional<String>,
    currentOwnerName: Optional<String>, nudgeCount: int, lastNudgedAt: Optional<Instant>,
    evalScore: Optional<Integer>, evalRationale: Optional<String>, status: TaskStatus,
    createdAt: Instant, completedAt: Optional<Instant>}. TaskStatus enum: CREATED, ASSIGNED,
    IN_PROGRESS, OVERDUE, NUDGED, ESCALATED, COMPLETED, CANCELLED. Events: TaskCreated,
    TaskAssigned, TaskAssignmentBlocked, TaskProgressStarted, TaskOverdue, NudgeSent,
    TaskEscalated, TaskCompleted, TaskCancelled, AssignmentEvalScored. Commands: createTask,
    assignTask, blockAssignment, markInProgress, markOverdue, recordNudge, escalate,
    complete, cancel, recordEval, getTask. emptyState() returns PlannerTask.initial("",
    null) without commandContext() reference.

- 1 Consumer BoardSyncConsumer subscribed to BoardEventQueue events; for each
  BoardEventReceived normalises the RawBoardEvent into a TaskSummary and either calls
  PlannerTaskEntity.createTask (for eventType "task.created") or
  PlannerTaskEntity.markInProgress / markOverdue (for eventType "task.updated" per the
  status field). On createTask success, starts a PlannerWorkflow with taskId as the workflow
  id.

- 1 View BoardView with row type BoardTaskRow (mirrors PlannerTask). Table updater consumes
  PlannerTaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM board_task_view. No
  WHERE status filter — caller filters client-side.

- 3 TimedActions:
  * PlannerPoller — every 20s, reads next line from src/main/resources/sample-events/
    board-events.jsonl and calls BoardEventQueue.receiveEvent.
  * StaleTaskChecker — every 5 minutes, queries BoardView.getAllTasks, picks OVERDUE or
    NUDGED tasks (oldest-first), calls NudgeAgent for each (up to 3 per tick), runs
    GuardrailChecker.checkNudge, then calls PlannerTaskEntity.recordNudge. Tasks with
    nudgeCount >= 3 get GuardrailChecker.checkEscalation then PlannerTaskEntity.escalate.
  * EvalRunner — every 60 minutes, queries BoardView.getAllTasks, picks up to 5 COMPLETED
    tasks without evalScore (oldest-first), calls EvalJudge, then calls
    PlannerTaskEntity.recordEval(score, rationale) per task.

- 2 HttpEndpoints:
  * BoardEndpoint at /api with GET /board, GET /board/{id}, POST /board/{id}/complete (body
    {resolvedBy}), POST /board/{id}/escalate (body {escalatedBy, reason}), GET /board/sse,
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Complete writes TaskCompleted; escalate writes
    TaskEscalated.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

- GuardrailChecker.java — a plain Java class (NOT an Akka component) called synchronously
  from both the workflow and StaleTaskChecker. Three static methods:
  DispatchResult checkAssignment(AssignmentRecommendation rec, TaskSummary task) — checks
    owner capacity (stubbed: always passes unless ownerId ends in "_FULL") and assignment
    authority (all owners may be assigned tasks in the simulator).
  DispatchResult checkNudge(NudgeDraft draft, PlannerTask task) — blocks if nudgeCount >= 3
    and tone != ESCALATORY, or if message content contains a forbidden phrase list.
  DispatchResult checkEscalation(String escalatedBy, PlannerTask task) — always passes in
    the simulator (authority check stubbed to true).

Companion files:
- PlannerTasks.java declaring three Task<R> constants: ASSIGN (AssignmentRecommendation),
  NUDGE (NudgeDraft), EVAL (EvalResult).
- Domain records RawBoardEvent, TaskSummary, AssignmentRecommendation, NudgeDraft,
  DispatchResult, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9822 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/board-events.jsonl with 10 canned board event lines
  covering task.created, task.updated with status changes, and a mix of priorities.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control: G1 guardrail before-tool-call.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with sector ops-automation, decisions.authority_level
  advisory-with-gate, oversight.human_on_loop = true (board is always visible),
  failure.failure_modes including "incorrect-assignment" and "nudge-to-wrong-person";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/assignment-agent.md, prompts/nudge-agent.md, prompts/eval-judge.md loaded as
  agent system prompts.
- README.md at the project root: title "Akka Sample: Project Manager Agent (Planner)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live task list sorted OVERDUE-first with status pills, priority badges, assignee
  chips; right = selected task detail with assignment rationale, nudge history, and
  complete/escalate buttons). Browser title exactly:
  <title>Akka Sample: Project Manager Agent (Planner)</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with per-agent dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry pseudo-randomly.
- Per-agent mock-response shapes for THIS blueprint:
    assignment-agent.json — 8–10 AssignmentRecommendation entries covering
      different team members, priorities, rationales. One entry should have
      ownerId ending in "_FULL" to trigger the guardrail block in J5.
    nudge-agent.json — 6 NudgeDraft entries spanning all three NudgeTone
      values with varying message lengths and phrasing styles.
    eval-judge.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales aligned to the rubric axes.
- MockModelProvider.seedFor(taskId) makes per-task selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run". (Lesson 1)
- Optional<T> for every nullable field on a View row record. (Lesson 4)
- WorkflowSettings is nested inside Workflow — no import needed. (Lesson 6)
- emptyState() never calls commandContext(). (Lesson 7)
- AutonomousAgent never silently downgraded to Agent. (Lesson 8)
- All agents in this blueprint are typed (Agent, not AutonomousAgent). (Lesson 9)
- GuardrailChecker is a plain Java class, not an Akka component. (Lesson 10)
- Before-tool-call hooks are synchronous checks on the calling thread. (Lesson 11)
- BoardSyncConsumer normalises events BEFORE starting workflows. (Lesson 12)
- PlannerWorkflow step timeouts: recommendStep 15s, guardrailCheckStep 5s,
  assignStep 10s. On timeout the workflow emits TaskAssignmentBlocked. (Lesson 13)
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. (Lesson 25)
- No forbidden words in user-facing text (Lesson 23).
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
