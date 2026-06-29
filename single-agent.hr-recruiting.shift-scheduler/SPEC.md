# SPEC — nexshift

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** NexShift.
**One-line pitch:** A manager submits an open-shift list and an employee roster; one AI agent fills the schedule by emitting `AssignShift` tool calls that are validated by a `before-tool-call` guardrail before any assignment is committed — preventing double-bookings, hour-limit breaches, and unqualified placements.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the hr-recruiting domain. One `NexShiftAgent` (AutonomousAgent) carries the entire scheduling decision; the surrounding components prepare its inputs, guard its tool calls, and persist the results. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** intercepts every `AssignShift` tool call the agent emits before the assignment lands on the entity. The guardrail checks: the target employee is available during the shift window, the employee does not already have a shift that overlaps, adding this shift would not push the employee over the weekly hour cap, and the employee holds a qualification required by the shift role. A rejected call returns a structured rejection to the agent loop so the agent can reassign to a different employee; the entity never sees invalid writes.

The blueprint shows that the single-agent pattern is not the same as "ungoverned" — one independent check sits on every schedule write.

## 3. User-facing flows

The user opens the App UI tab.

1. The manager picks a **date range** (e.g., the coming work week) and a **department** from the dropdown.
2. The manager clicks **Generate schedule**. The UI POSTs to `/api/schedules` and receives a `scheduleId`.
3. The schedule card appears in the live list in `PENDING` state. Within ~1 s it transitions to `BUILDING` — the roster of available employees has been fetched and summarized.
4. Within ~10–60 s, the workflow's `scheduleStep` completes. The card transitions to `SCHEDULING` then `DRAFT`. The draft schedule appears: a table of shifts with the assigned employee, role, start and end time, and a status chip (`ASSIGNED` or `GUARDRAIL_BLOCKED`).
5. The manager reviews the draft and clicks **Confirm schedule**. The UI PUTs to `/api/schedules/{id}/confirm`. The card transitions to `CONFIRMED`.
6. The manager can submit another scheduling run; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ScheduleEndpoint` | `HttpEndpoint` | `/api/schedules/*` — submit, list, get, confirm, SSE; serves `/api/metadata/*`. | — | `ScheduleEntity`, `ScheduleView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ScheduleEntity` | `EventSourcedEntity` | Per-run lifecycle: pending → building → scheduling → draft → confirmed. Source of truth. | `ScheduleEndpoint`, `ScheduleWorkflow` | `ScheduleView` |
| `ScheduleWorkflow` | `Workflow` | One workflow per scheduleId. Steps: `buildRosterStep` → `scheduleStep` → `confirmStep`. | started by `ScheduleEndpoint` once entity is created | `NexShiftAgent`, `ScheduleEntity` |
| `NexShiftAgent` | `AutonomousAgent` | The one decision-making LLM. Receives open shifts and employee roster in the task definition; emits `AssignShift` tool calls to fill slots. Stops when all shifts are filled or the agent exhausts its iteration budget. | invoked by `ScheduleWorkflow` | `AssignmentGuardrail` → `ScheduleEntity` |
| `AssignmentGuardrail` | supporting class | Registered on `NexShiftAgent` via the `before-tool-call` hook. Validates every `AssignShift` call before it reaches the entity. | NexShiftAgent tool calls | accepts or rejects each call |
| `ScheduleView` | `View` | Read model: one row per schedule run for the UI. | `ScheduleEntity` events | `ScheduleEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Shift(
    String shiftId,
    String role,           // e.g. "Nurse", "Technician", "Coordinator"
    Instant startTime,
    Instant endTime,
    String department,
    List<String> requiredQualifications
) {}

record Employee(
    String employeeId,
    String name,
    List<String> qualifications,
    List<AvailabilityWindow> availability,
    int weeklyHoursCap,
    int hoursScheduledThisWeek
) {}

record AvailabilityWindow(
    DayOfWeek day,
    LocalTime from,
    LocalTime to
) {}

record ShiftAssignment(
    String shiftId,
    String employeeId,
    AssignmentStatus status,
    Optional<String> blockedReason
) {}
enum AssignmentStatus { ASSIGNED, GUARDRAIL_BLOCKED, UNASSIGNED }

record ScheduleRequest(
    String scheduleId,
    LocalDate weekStart,
    String department,
    List<Shift> openShifts,
    List<Employee> employees,
    String submittedBy,
    Instant submittedAt
) {}

record ScheduleDraft(
    List<ShiftAssignment> assignments,
    int totalShifts,
    int filledCount,
    int blockedCount
) {}

record Schedule(
    String scheduleId,
    Optional<ScheduleRequest> request,
    Optional<ScheduleDraft> draft,
    ScheduleStatus status,
    Instant createdAt,
    Optional<Instant> confirmedAt
) {}

enum ScheduleStatus {
    PENDING, BUILDING, SCHEDULING, DRAFT, CONFIRMED, FAILED
}
```

Events on `ScheduleEntity`: `ScheduleRequested`, `RosterBuilt`, `SchedulingStarted`, `DraftRecorded`, `ScheduleConfirmed`, `ScheduleFailed`.

Every nullable lifecycle field on the `Schedule` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/schedules` — body `{ weekStart, department, openShifts: [Shift], employees: [Employee], submittedBy }` → `{ scheduleId }`.
- `GET /api/schedules` — list all schedule runs, newest-first.
- `GET /api/schedules/{id}` — one schedule run.
- `PUT /api/schedules/{id}/confirm` — confirm the draft; transitions to CONFIRMED.
- `GET /api/schedules/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: NexShift</title>`.

The App UI tab is a two-column layout: a left rail with the live list of schedule runs (status pill + filled/total count + department + age) and a right pane with the selected run's detail — open shift list, employee roster summary, assignment table (shift, employee, start/end, status chip, blocked reason if any), and a **Confirm schedule** button when status is `DRAFT`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `AssignShift` tool call the `NexShiftAgent` emits. Checks: (1) the employee is available (availability windows cover the shift start/end), (2) no overlapping shift already assigned to that employee in this run, (3) adding this shift's hours would not exceed `employee.weeklyHoursCap`, (4) the employee's `qualifications` include all `shift.requiredQualifications`. On any failure returns a structured `invalid-tool-call` rejection naming the failed check; the agent loop receives the rejection as tool output and may attempt a different employee. Passing calls are forwarded to `ScheduleEntity.recordAssignment(assignment)`.

## 9. Agent prompts

- `NexShiftAgent` → `prompts/nexshift-agent.md`. The single decision-making LLM. System prompt instructs it to iterate through open shifts in priority order, call `AssignShift` for each, handle rejections by reassigning to the next eligible employee, and stop when all shifts are filled or the iteration budget is exhausted.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Manager submits a 5-shift open week for the `Nursing` department; within 60 s all 5 shifts are assigned, no constraint violations, draft appears in the UI.
2. **J2** — The agent attempts to double-book an employee; the `before-tool-call` guardrail rejects the call; the agent reassigns to the next eligible employee; the draft shows `ASSIGNED` for that shift.
3. **J3** — An employee at their weekly hour cap receives zero new assignments; every attempt is blocked with reason `hour-cap-exceeded`; the guardrail audit log confirms the blocks.
4. **J4** — A shift requiring qualification `cert-BLS` is never assigned to an employee whose `qualifications` list omits it.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named nexshift demonstrating the single-agent × hr-recruiting cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-hr-recruiting-shift-scheduler. Java package io.akka.samples.nexshiftagent.
Akka 3.6.0. HTTP port 9451.

Components to wire (exactly):

- 1 AutonomousAgent NexShiftAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/nexshift-agent.md>) and
  .capability(TaskAcceptance.of(ASSIGN_SHIFTS).maxIterationsPerTask(20))
  (20 iterations to allow multi-pass filling of a full week's roster). The task receives
  the open shift list and employee roster as its instruction text; the agent emits
  AssignShift tool calls to assign employees to shifts. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the tool call returns a structured
  error as its tool result; the agent reads the rejection and retries with a different
  employee within its iteration budget.

- 1 Workflow ScheduleWorkflow per scheduleId with three steps:
  * buildRosterStep — reads the ScheduleEntity to get the submitted ScheduleRequest;
    summarizes employee availability into the format the agent prompt expects; emits
    RosterBuilt. WorkflowSettings.stepTimeout 10s.
  * scheduleStep — emits SchedulingStarted, then calls componentClient.forAutonomousAgent(
    NexShiftAgent.class, "scheduler-" + scheduleId).runSingleTask(
      TaskDef.instructions(formatRosterAndShifts(request))
    ) and polls until the task finishes. On completion calls
    ScheduleEntity.recordDraft(draft). WorkflowSettings.stepTimeout 120s with
    defaultStepRecovery maxRetries(1).failoverTo(ScheduleWorkflow::error).
  * confirmStep — no-op step that waits for the explicit confirm command; entity handles
    the CONFIRMED transition. WorkflowSettings.stepTimeout 3600s (manager must confirm
    within 1 h or the run expires to FAILED).

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ScheduleEntity (one per scheduleId). State Schedule{scheduleId: String,
  request: Optional<ScheduleRequest>, draft: Optional<ScheduleDraft>,
  status: ScheduleStatus, createdAt: Instant, confirmedAt: Optional<Instant>}.
  ScheduleStatus enum: PENDING, BUILDING, SCHEDULING, DRAFT, CONFIRMED, FAILED.
  Events: ScheduleRequested{request}, RosterBuilt{}, SchedulingStarted{},
  DraftRecorded{draft}, ScheduleConfirmed{confirmedAt: Instant}, ScheduleFailed{reason}.
  Commands: submit, markBuilding, markScheduling, recordDraft, confirm, fail, getSchedule.
  emptyState() returns Schedule.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) in
  event-appliers.

- 1 View ScheduleView with row type ScheduleRow (mirrors Schedule). Table updater consumes
  ScheduleEntity events. ONE query getAllSchedules: SELECT * AS schedules FROM schedule_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ScheduleEndpoint at /api with POST /schedules (body
    {weekStart, department, openShifts: [Shift], employees: [Employee], submittedBy};
    mints scheduleId; calls ScheduleEntity.submit; starts ScheduleWorkflow; returns
    {scheduleId}), GET /schedules (list from getAllSchedules, sorted newest-first),
    GET /schedules/{id} (one row), PUT /schedules/{id}/confirm (calls
    ScheduleEntity.confirm), GET /schedules/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ScheduleTasks.java declaring one Task<R> constant: ASSIGN_SHIFTS = Task.name("Assign
  shifts").description("Fill the open shift list by calling AssignShift for each slot;
  handle rejections by trying the next eligible employee").resultConformsTo(
  ScheduleDraft.class). DO NOT skip this — the AutonomousAgent requires its companion Tasks
  class (Lesson 7).

- Domain records Shift, Employee, AvailabilityWindow, ShiftAssignment, AssignmentStatus,
  ScheduleRequest, ScheduleDraft, Schedule, ScheduleStatus.

- AssignmentGuardrail.java implementing the before-tool-call hook. Receives the candidate
  AssignShift tool call, runs the four checks listed in eval-matrix.yaml G1, and either
  passes the call through to ScheduleEntity.recordAssignment or returns
  Guardrail.reject(<structured-error>) naming the failed check so the agent can retry
  with a different employee.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9451 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The NexShiftAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/shifts.jsonl with 3 seeded shift sets:
  a 5-shift nursing week, a 7-shift technician week, and a 4-shift coordinator week.
  Each shift has a role, start/end time, department, and 1–2 required qualifications.

- src/main/resources/sample-events/employees.jsonl with 6 seeded employees covering the
  three departments with varied qualifications, availability windows, and weekly hour caps.
  Two employees per department: one whose schedule is nearly full (tests the hour-cap
  guardrail) and one who is fully available.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.sector = hr-scheduling,
  decisions.authority_level = draft-for-human-approval (the agent's schedule is a draft
  the manager confirms, not an enforced assignment), oversight.human_in_loop = true
  (the manager confirms before the schedule is live), failure.failure_modes including
  "double-booking", "hour-cap-breach", "unqualified-placement",
  "shift-left-unassigned-silently"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/nexshift-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: NexShift Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of schedule runs; right = selected-run detail with open shift list, employee
  roster summary, assignment table, and Confirm button when status is DRAFT).
  Browser title exactly: <title>Akka Sample: NexShift</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(scheduleId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    assign-shifts.json — 6 ScheduleDraft entries covering a range of outcomes:
      fully filled, partially filled, one blocked assignment per run. Each entry's
      assignments list covers every shift in the seeded dataset for the matched department.
      Two entries include at least one ShiftAssignment with status GUARDRAIL_BLOCKED
      and a non-empty blockedReason ("double-booking" / "hour-cap-exceeded") to exercise
      the guardrail path. The mock selects the blocked entry on the first call of every
      even-indexed schedule run (modulo seed) so J2/J3 are reproducible.
- A MockModelProvider.seedFor(scheduleId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. NexShiftAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ScheduleTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (buildRosterStep
  10s, scheduleStep 120s, confirmStep 3600s, error 5s).
- Lesson 6: every nullable lifecycle field on the Schedule row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ScheduleTasks.java with ASSIGN_SHIFTS = Task.name(...).description(...)
  .resultConformsTo(ScheduleDraft.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9451 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (NexShiftAgent).
  The before-tool-call guardrail (AssignmentGuardrail) is NOT a second agent.
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
