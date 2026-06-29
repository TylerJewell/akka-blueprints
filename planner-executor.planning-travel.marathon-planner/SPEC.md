# SPEC — marathon-planner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Marathon Training Planner.
**One-line pitch:** Submit a runner profile and goal race; a TrainingPlannerAgent assesses baseline fitness, builds a periodized plan on a plan ledger, dispatches each training phase to a WorkoutBuilderAgent, scores every adjustment against an adherence signal, and replans when the runner's goal drifts out of reach.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with dynamic skill loading and MCP tool mapping. The TrainingPlannerAgent owns two ledgers — a **plan ledger** (runner goal, baseline facts, phase structure, current dispatch) and an **adjustment ledger** (every plan-adjustment attempt, adherence score, revision rationale). On each loop iteration the TrainingPlannerAgent reads both ledgers, picks the next executor role (assess, build, validate, or compose), and either continues, replans on low adherence, or completes.

The blueprint also demonstrates two governance mechanisms wired into that loop:

- a **before-tool-invocation guardrail** that checks each skill dispatch against the capability registry before the agent runs,
- an **on-decision eval-event** that scores every plan adjustment against the runner's stated goal and logs the result before the next loop tick.

## 3. User-facing flows

The user opens the App UI tab and submits a runner profile via the form.

1. The system creates a `RunnerPlan` record in `ASSESSING` and starts a `PlanWorkflow`.
2. The FitnessAssessorAgent evaluates the incoming `RunnerProfile` and produces a `FitnessBaseline`. The plan moves to `PLANNING`.
3. The TrainingPlannerAgent drafts a `PlanLedger { goalFacts, missingFacts, phases, currentDispatch }` and emits `PlanDrafted`.
4. The workflow enters the executor loop. Each iteration:
   - TrainingPlannerAgent reads both ledgers and proposes a `SkillDispatch { skill, task, rationale }`.
   - The **before-tool-invocation guardrail** checks the proposed skill against `SkillRegistryEntity`; on rejection the workflow records a `WorkoutBlocked` entry and asks the planner to revise.
   - The WorkoutBuilderAgent runs the task and returns a typed `WorkoutResult`.
   - The **adherence eval-event** scores the result against the plan goal and appends an `AdjustmentEntry { score, verdict, notes }` to the adjustment ledger.
   - On a score below the adherence threshold, the workflow records a `LowAdherenceSignal` and asks the TrainingPlannerAgent to replan.
5. The TrainingPlannerAgent decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After two consecutive replans without adherence improvement, or three consecutive failures on the same task, the planner emits `FAIL`.
6. On `COMPLETE`, the TrainingPlannerAgent produces a `PlanSummary { overview, weeklySchedule, nutritionNotes }` and emits `PlanCompleted`. The RunnerPlan moves to `COMPLETED`.

A `PlanSimulator` (TimedAction) drips a sample runner profile every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TrainingPlannerAgent` | `AutonomousAgent` | Drafts and adjusts the plan ledger; decides next step on each loop tick. Produces `PlanSummary` on completion. | `PlanWorkflow` | returns typed result to workflow |
| `WorkoutBuilderAgent` | `AutonomousAgent` | Expands a phase stub into full workout sessions for one week. Returns `WorkoutResult`. | `PlanWorkflow` | — |
| `FitnessAssessorAgent` | `AutonomousAgent` | Evaluates runner profile data and fixture baselines; returns `FitnessBaseline`. | `PlanWorkflow` | — |
| `PlanWorkflow` | `Workflow` | Drives the assess → plan → guardrail → build → eval → decide loop, plus replan and fail branches. | `PlanEndpoint`, `PlanRequestConsumer` | `RunnerEntity`, `SkillRegistryEntity` |
| `RunnerEntity` | `EventSourcedEntity` | Holds the runner's profile, plan ledger, adjustment ledger, and final summary. | `PlanWorkflow` | `PlanView` |
| `SkillRegistryEntity` | `EventSourcedEntity` | Holds the current set of loaded skill capabilities. Single instance keyed by literal `"registry"`. | `PlanEndpoint` (skill management) | `PlanWorkflow` (guardrail reads) |
| `PlanQueue` | `EventSourcedEntity` | Audit log of submitted plan requests. | `PlanEndpoint`, `PlanSimulator` | `PlanRequestConsumer` |
| `PlanView` | `View` | List-of-plans read model for the UI. | `RunnerEntity` events | `PlanEndpoint` |
| `PlanRequestConsumer` | `Consumer` | Subscribes to `PlanQueue` events; starts a `PlanWorkflow` per submission. | `PlanQueue` events | `PlanWorkflow` |
| `PlanSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/runner-profiles.jsonl` and enqueues it. | scheduler | `PlanQueue` |
| `StaleWorkoutMonitor` | `TimedAction` | Every 30 s, marks any plan stuck in `BUILDING` past 5 minutes as `STALE`. | scheduler | `RunnerEntity` |
| `PlanEndpoint` | `HttpEndpoint` | `/api/plans/*` — submit, get, list, SSE, skill management. | — | `PlanView`, `PlanQueue`, `RunnerEntity`, `SkillRegistryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record RunnerProfile(
    String runnerId,
    String name,
    String goalRace,
    LocalDate goalRaceDate,
    int targetFinishMinutes,
    double currentWeeklyMiles,
    List<String> injuryHistory
) {}

record FitnessBaseline(
    double estimatedVo2Max,
    double currentLongRunMiles,
    double currentWeeklyMiles,
    Optional<String> recentRaceResult,
    String fitnessLevel
) {}

record PlanPhase(
    String phaseName,
    int durationWeeks,
    double targetWeeklyMiles,
    List<String> keyWorkoutTypes
) {}

record PlanLedger(
    List<String> goalFacts,
    List<String> missingFacts,
    List<PlanPhase> phases,
    Optional<SkillDispatch> currentDispatch
) {}

record SkillDispatch(
    SkillKind skill,
    String task,
    String rationale
) {}

record WorkoutSession(
    String sessionId,
    String dayOfWeek,
    WorkoutType workoutType,
    double distanceMiles,
    int targetPaceSecondsPerMile,
    String notes
) {}

record WorkoutResult(
    SkillKind skill,
    String task,
    boolean ok,
    List<WorkoutSession> sessions,
    Optional<String> errorReason
) {}

record AdjustmentEntry(
    int attempt,
    SkillKind skill,
    String task,
    AdjustmentVerdict verdict,
    double adherenceScore,
    String notes,
    Optional<String> blocker,
    Instant recordedAt
) {}

record AdjustmentLedger(List<AdjustmentEntry> entries) {}

record PlanSummary(
    String overview,
    List<WorkoutSession> weeklySchedule,
    List<String> nutritionNotes,
    Instant producedAt
) {}

record RunnerPlan(
    String planId,
    String runnerId,
    String goalRace,
    RunnerPlanStatus status,
    Optional<FitnessBaseline> baseline,
    Optional<PlanLedger> ledger,
    Optional<AdjustmentLedger> adjustments,
    Optional<PlanSummary> summary,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SkillKind { ASSESS, BUILD_WORKOUTS, VALIDATE_PLAN, COMPOSE_SUMMARY }
enum AdjustmentVerdict { OK, LOW_ADHERENCE, BLOCKED_BY_REGISTRY, FAILED }
enum WorkoutType { EASY_RUN, LONG_RUN, TEMPO, INTERVAL, RECOVERY, CROSS_TRAIN }
enum RunnerPlanStatus { ASSESSING, PLANNING, BUILDING, COMPLETED, FAILED, STALE }
```

### Events (`RunnerEntity`)

`RunnerPlanCreated`, `BaselineAssessed`, `PlanDrafted`, `WorkoutBuilt`, `WorkoutBlocked`, `LedgerRevised`, `AdjustmentRecorded`, `PlanCompleted`, `PlanFailed`, `PlanFailedTimeout`.

### Events (`SkillRegistryEntity`)

`SkillLoaded { skillId, name, loadedAt }`, `SkillUnloaded { skillId, unloadedAt }`.

### Events (`PlanQueue`)

`PlanRequested { planId, runnerId, goalRace, goalRaceDate, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/plans` — body `{ runnerId, name, goalRace, goalRaceDate, targetFinishMinutes, currentWeeklyMiles, injuryHistory? }` → `202 { planId }`.
- `GET /api/plans` — list all plans. Optional `?status=...`.
- `GET /api/plans/{id}` — one plan (full ledger + adjustments + summary).
- `GET /api/plans/sse` — server-sent events stream of every plan change.
- `POST /api/skills/load` — body `{ skillId, name }` → `200`. Adds a skill to the registry.
- `DELETE /api/skills/{skillId}` — removes a skill from the registry.
- `GET /api/skills` — `{ skills: [{ skillId, name, loadedAt }] }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Marathon Training Planner"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a runner profile, skill registry management panel, live list of plans with status pills, expand-row to see the plan ledger, the adjustment ledger entries, and the final plan summary.

Browser title: `<title>Akka Sample: Marathon Training Planner</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-invocation guardrail** (`before-tool-call` on `TrainingPlannerAgent`): every `SkillDispatch` is checked against the `SkillRegistryEntity` before the target agent runs. If the requested `SkillKind` is not present in the registry, the workflow records a `WorkoutBlocked` entry and returns the block reason to the planner for revision. Blocking. Failure → `AdjustmentEntry` with `verdict = BLOCKED_BY_REGISTRY` + replan request.
- **E1 — on-decision eval-event** (`eval-event`, flavor `on-decision-eval`): after every `WorkoutResult` is received, an `AdherenceEvaluator` scores the result against the plan goal (target finish time, phase mileage, workout type mix). The score is appended as an `AdjustmentEntry.adherenceScore` (0.0–1.0). Scores below the configured threshold (default 0.6) trigger a `LowAdherenceSignal` that causes the planner to replan the affected phase.

## 9. Agent prompts

- `TrainingPlannerAgent` → `prompts/training-planner.md`. Owns both ledgers; decides next step.
- `WorkoutBuilderAgent` → `prompts/workout-builder.md`. Builds weekly sessions from phase stubs.
- `FitnessAssessorAgent` → `prompts/fitness-assessor.md`. Evaluates runner profile; returns baseline.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a runner profile targeting a full marathon in 20 weeks. Plan progresses `ASSESSING → PLANNING → BUILDING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. Expanded view shows a plan ledger with 3–5 phases, an adjustment ledger with 3–10 entries, and a `PlanSummary` with a weekly schedule.
2. **J2** — Load a skill not present in the registry (`SkillKind.UNKNOWN`). The guardrail blocks the dispatch; a `WorkoutBlocked` entry appears with `verdict = BLOCKED_BY_REGISTRY`. The planner revises to a registered skill.
3. **J3** — Submit a runner profile where the adherence evaluator scores the first workout build below 0.6. A `LowAdherenceSignal` fires; the planner replans the affected phase. The adjustment ledger shows the low-score entry followed by a revised build with a higher score.
4. **J4** — Submit a profile with a goal race date fewer than 8 weeks away. The planner emits `PlanFailed` with `failureReason = "goal race date too close for safe periodization"` rather than generating an unsafe plan.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named marathon-planner demonstrating the
planner-executor × planning-travel cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
planner-executor-planning-travel-marathon-planner. Java package
io.akka.samples.marathontrainingplanner. Akka 3.6.0. HTTP port 9603.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TrainingPlannerAgent — definition() with capabilities:
      capability(TaskAcceptance.of(DRAFT_PLAN).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(DECIDE_NEXT).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(COMPOSE_SUMMARY).maxIterationsPerTask(2)).
    System prompt from prompts/training-planner.md. DRAFT_PLAN returns PlanLedger.
    DECIDE_NEXT returns a PlanStep tagged union (Continue(SkillDispatch) |
    Replan(PlanLedger) | Complete(PlanSummary stub) | Fail(String reason)).
    COMPOSE_SUMMARY returns PlanSummary.
  * WorkoutBuilderAgent — capability(TaskAcceptance.of(BUILD_WORKOUTS)
      .maxIterationsPerTask(2)).
    Prompt from prompts/workout-builder.md. Returns WorkoutResult.
  * FitnessAssessorAgent — capability(TaskAcceptance.of(ASSESS)
      .maxIterationsPerTask(2)).
    Prompt from prompts/fitness-assessor.md. Returns FitnessBaseline.

- 1 Workflow PlanWorkflow with steps:
  assessStep -> planStep -> [loop entry] guardStep -> dispatchStep ->
  evalStep -> decideStep -> [back to guardStep or to summariseStep /
  failStep / staleStep].
  Step timeouts (Lesson 4):
    assessStep ofSeconds(60), planStep ofSeconds(60), guardStep
    ofSeconds(10), dispatchStep ofSeconds(120), evalStep ofSeconds(30),
    decideStep ofSeconds(45), summariseStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(PlanWorkflow::error)).
  guardStep reads SkillRegistryEntity.get("registry"); on missing skill records
  a WorkoutBlocked entry via RunnerEntity.recordBlock(task, reason) and loops
  back to decideStep so the planner sees the blocker.
  dispatchStep uses switch on SkillDispatch.skill to call WorkoutBuilderAgent
  (BUILD_WORKOUTS) or FitnessAssessorAgent (ASSESS) via
  forAutonomousAgent(...).runSingleTask(...).
  evalStep runs AdherenceEvaluator.score(WorkoutResult, PlanLedger) and
  appends the AdjustmentEntry via RunnerEntity.recordAdjustment(entry). On
  score < threshold emits LowAdherenceSignal to entity and loops to decideStep
  with the signal in context.
  decideStep calls forAutonomousAgent(TrainingPlannerAgent.class, DECIDE_NEXT);
  on Continue or Replan loops; on Complete transitions to summariseStep ->
  completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity RunnerEntity holding RunnerPlan state. emptyState()
  returns RunnerPlan.initial() with status ASSESSING. Commands: createPlan,
  recordBaseline, recordPlanDraft, recordWorkout, recordBlock,
  recordAdjustment, reviseLedger, completePlan, failPlan, timeoutFail,
  getPlan. Events as listed in SPEC §5.

- 1 EventSourcedEntity SkillRegistryEntity keyed by literal "registry". State
  SkillRegistry { Map<String,SkillCapability> skills }. Commands: loadSkill,
  unloadSkill, getRegistry. Events: SkillLoaded, SkillUnloaded.

- 1 EventSourcedEntity PlanQueue with command enqueuePlan(planId, runnerId,
  goalRace, goalRaceDate) emitting PlanRequested.

- 1 View PlanView with row type PlanRow (mirror of RunnerPlan minus heavy
  ledger payloads — truncate to last 3 adjustment entries plus counts; the UI
  fetches the full plan by id on click). Table updater consumes RunnerEntity
  events. ONE query getAllPlans SELECT * AS plans FROM plan_view. No WHERE
  status filter — caller filters client-side (Lesson 2).

- 1 Consumer PlanRequestConsumer subscribed to PlanQueue events; on
  PlanRequested starts a PlanWorkflow with planId as the workflow id.

- 2 TimedActions:
  * PlanSimulator — every 90s, reads next line from
    src/main/resources/sample-events/runner-profiles.jsonl and calls
    PlanQueue.enqueuePlan.
  * StaleWorkoutMonitor — every 30s, queries PlanView.getAllPlans, filters
    BUILDING plans whose createdAt is older than 5 minutes, calls
    RunnerEntity.timeoutFail; PlanWorkflow polls RunnerEntity.getPlan in its
    decideStep and exits when status == STALE.

- 2 HttpEndpoints:
  * PlanEndpoint at /api with POST /plans, GET /plans (filters client-side
    from getAllPlans), GET /plans/{id}, GET /plans/sse,
    POST /skills/load, DELETE /skills/{skillId}, GET /skills, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: DRAFT_PLAN
  (resultConformsTo PlanLedger), DECIDE_NEXT (PlanStep), COMPOSE_SUMMARY
  (PlanSummary).
- ExecutorTasks.java declaring two Task<R> constants: BUILD_WORKOUTS,
  ASSESS (all resultConformsTo WorkoutResult and FitnessBaseline
  respectively).
- Domain records as listed in SPEC §5, plus a PlanStep sealed interface
  with permits Continue(SkillDispatch), Replan(PlanLedger revised),
  Complete(PlanSummary stub), Fail(String reason).
- application/AdherenceEvaluator.java — deterministic scorer. Inputs:
  WorkoutResult + PlanLedger. Score 0.0–1.0 based on: (a) workout type
  mix vs. phase's keyWorkoutTypes, (b) total weekly mileage within 10%
  of phase targetWeeklyMiles, (c) all sessions have valid paceSeconds.
  Returns AdjustmentEntry with score and verdict.
- application/CapabilityGuardrail.java — deterministic vetter. Reject if
  SkillDispatch.skill is not present in the SkillRegistryEntity snapshot.
  Reject if BUILD_WORKOUTS task requests more than 20 sessions per week.
  Reject if ASSESS task is dispatched after baseline is already set.
  Returns a GuardrailResult { allowed, reason }.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9603 and agent model-provider blocks
  for anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/runner-profiles.jsonl with 8 canned
  runner profiles (varied goals: 5 K, half marathon, full marathon;
  varied fitness levels; varied injury histories).
- src/main/resources/sample-data/fitness-fixtures.jsonl — 12 canned
  fitness assessments keyed by fitnessLevel (beginner/intermediate/
  advanced/elite). Used by FitnessAssessorAgent.
- src/main/resources/sample-data/workout-templates.jsonl — 16 weekly
  workout templates keyed by (phaseType, fitnessLevel). Used by
  WorkoutBuilderAgent.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies for the metadata endpoint).
- eval-matrix.yaml at the project root with 2 controls (G1, E1).
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, compliance for the
  training-planner domain.
- prompts/training-planner.md, prompts/workout-builder.md,
  prompts/fitness-assessor.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Marathon Training
  Planner", one-line pitch, prerequisites (integration form: None),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license.
- src/main/resources/static-resources/index.html — single self-contained
  HTML file. Five tabs: Overview, Architecture (4 mermaid diagrams +
  comp-row table), Risk Survey (7 sub-tabs), Eval Matrix (5-column table),
  App UI (form + skill registry panel + live plan list with status pills
  and expand-on-click for ledgers and summary). Browser title exactly:
  <title>Akka Sample: Marathon Training Planner</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java per-agent
  dispatch on agent class name and Task<R> id. Each branch reads a JSON
  file from src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    training-planner.json — three lists keyed by task id:
      "DRAFT_PLAN" → 4 PlanLedger entries with 3–5 phases each.
      "DECIDE_NEXT" → 5 PlanStep entries cycling Continue / Replan /
        Complete across the three executor skills.
      "COMPOSE_SUMMARY" → 4 PlanSummary entries with 60–120 word
        overviews, 5–7 weekly sessions, 3 nutrition bullets.
    workout-builder.json — 6 WorkoutResult entries, ok=true, sessions
      fields with 5–7 WorkoutSession records covering a full week.
    fitness-assessor.json — 6 FitnessBaseline entries spanning
      beginner → elite fitness levels with varied Vo2Max estimates.

Constraints:
- Lesson 1: AutonomousAgent never silently downgraded — extends
  AutonomousAgent for every agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on assessStep,
  planStep, dispatchStep, evalStep, decideStep, summariseStep.
- Lesson 6: Optional<T> for every nullable field on PlanRow and
  RunnerPlan entity state.
- Lesson 7: Companion PlannerTasks.java and ExecutorTasks.java declaring
  every Task<R> constant.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9603 in application.conf.
- Lesson 11: source.platform never in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names.
- Lesson 24: index.html includes mermaid CSS overrides AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching by data-tab / data-panel attribute; no zombie
  panels.
- Overview tab's Try-it card shows just "/akka:build".
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
