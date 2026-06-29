# SPEC — energy-management-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Energy Efficiency Management Agent.
**One-line pitch:** Trigger an optimization cycle; a coordinator dispatches HVAC, lighting, and equipment analysis to three specialists in parallel, gates every control action through a safety guardrail, and consolidates results into one efficiency report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their subsystem reports, and asks a fourth AutonomousAgent to consolidate a unified efficiency report. The blueprint also demonstrates a **before-tool-call guardrail** that gates every building control action before it executes, and an **eval-event** that samples a completed cycle's consolidated report for an operational quality score.

## 3. User-facing flows

The user opens the App UI tab and triggers an optimization cycle for a building zone via the form.

1. The system creates an `OptimizationCycle` record in `QUEUED` and starts an `OptimizationWorkflow`.
2. The Coordinator decomposes the cycle into three parallel work items: an HVAC analysis task, a lighting analysis task, and an equipment analysis task.
3. The workflow forks: all three specialists run concurrently. Each returns a typed subsystem report.
4. The Coordinator consolidates the three reports into a `ConsolidatedReport { summary, hvacRecommendations, lightingRecommendations, equipmentRecommendations, estimatedSavingsKwh, guardrailVerdict }`.
5. Before any recommended control action executes, a before-tool-call guardrail checks whether the action stays within safety and comfort limits. Actions that fail are logged as `BlockedAction` events and excluded from the report.
6. The cycle moves to `COMPLETE`. If any specialist times out after 90 seconds, the workflow produces a `PARTIAL` cycle using whichever reports arrived.

A `TelemetrySimulator` (TimedAction) drips a sample building telemetry event every 90 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EnergyCoordinator` | `AutonomousAgent` | Decomposes the cycle into subsystem tasks; consolidates the three specialist reports into one efficiency report. | `OptimizationWorkflow` | returns typed result to workflow |
| `HvacSpecialist` | `AutonomousAgent` | Analyzes HVAC telemetry; returns setpoint and schedule recommendations. Before-tool-call guardrail gates every control action call. | `OptimizationWorkflow` | — |
| `LightingSpecialist` | `AutonomousAgent` | Analyzes occupancy + daylight data; returns zone lighting schedule recommendations. Before-tool-call guardrail gates every control action call. | `OptimizationWorkflow` | — |
| `EquipmentSpecialist` | `AutonomousAgent` | Identifies equipment running outside efficient operating bands; recommends load-shifting or setback. Before-tool-call guardrail gates every control action call. | `OptimizationWorkflow` | — |
| `OptimizationWorkflow` | `Workflow` | Coordinates the parallel fan-out, consolidation, and blocked-action logging. | `CycleEndpoint`, `TelemetryConsumer` | `OptimizationCycleEntity` |
| `OptimizationCycleEntity` | `EventSourcedEntity` | Holds the cycle's lifecycle (queued → dispatched → consolidating → complete / partial / blocked). | `OptimizationWorkflow` | `CycleView` |
| `TelemetryQueue` | `EventSourcedEntity` | Logs each incoming telemetry event for replay and audit. | `CycleEndpoint`, `TelemetrySimulator` | `TelemetryConsumer` |
| `CycleView` | `View` | List-of-cycles read model. | `OptimizationCycleEntity` events | `CycleEndpoint` |
| `TelemetryConsumer` | `Consumer` | Listens to `TelemetryQueue` events and starts one `OptimizationWorkflow` per telemetry event. | `TelemetryQueue` events | `OptimizationWorkflow` |
| `TelemetrySimulator` | `TimedAction` | Drips a sample telemetry event every 90 s. | scheduler | `TelemetryQueue` |
| `EfficiencyEvalSampler` | `TimedAction` | Samples one `COMPLETE` cycle every 10 minutes for eval scoring; emits an `EvalScored` event. | scheduler | `OptimizationCycleEntity` |
| `CycleEndpoint` | `HttpEndpoint` | `/api/cycles/*` — trigger, get, list, SSE. | — | `CycleView`, `TelemetryQueue`, `OptimizationCycleEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TelemetryEvent(String buildingId, String zone, Instant recordedAt) {}

record HvacReport(
    List<SetpointRecommendation> recommendations,
    double currentLoadKw,
    Instant analysedAt
) {}
record SetpointRecommendation(String unit, String parameter, double currentValue,
                              double recommendedValue, String rationale) {}

record LightingReport(
    List<LightingScheduleEntry> scheduleChanges,
    double currentLoadKw,
    Instant analysedAt
) {}
record LightingScheduleEntry(String zone, String startTime, String endTime,
                             int targetLuxLevel, String rationale) {}

record EquipmentReport(
    List<EquipmentAction> actions,
    double currentLoadKw,
    Instant analysedAt
) {}
record EquipmentAction(String assetId, String actionType, String rationale,
                       boolean blockedByGuardrail) {}

record OptimizationPlan(
    String hvacQuery,
    String lightingQuery,
    String equipmentQuery
) {}

record ConsolidatedReport(
    String summary,
    HvacReport hvacRecommendations,
    LightingReport lightingRecommendations,
    EquipmentReport equipmentRecommendations,
    double estimatedSavingsKwh,
    String guardrailVerdict,
    Instant consolidatedAt
) {}

record OptimizationCycle(
    String cycleId,
    String buildingId,
    String zone,
    CycleStatus status,
    Optional<HvacReport> hvacReport,
    Optional<LightingReport> lightingReport,
    Optional<EquipmentReport> equipmentReport,
    Optional<ConsolidatedReport> consolidated,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant startedAt,
    Optional<Instant> finishedAt
) {}

enum CycleStatus { QUEUED, DISPATCHED, CONSOLIDATING, COMPLETE, PARTIAL, BLOCKED }
```

### Events (on `OptimizationCycleEntity`)

`CycleStarted`, `HvacReportAttached`, `LightingReportAttached`, `EquipmentReportAttached`, `CycleConsolidated`, `CyclePartial`, `CycleBlocked`, `EvalScored`.

### Events (on `TelemetryQueue`)

`TelemetryReceived { cycleId, buildingId, zone, recordedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/cycles` — body `{ buildingId, zone }` → `{ cycleId }`. Starts a workflow.
- `GET /api/cycles` — list all cycles. Optional `?status=QUEUED|DISPATCHED|CONSOLIDATING|COMPLETE|PARTIAL|BLOCKED`.
- `GET /api/cycles/{id}` — one cycle.
- `GET /api/cycles/sse` — server-sent events stream of every cycle change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Energy Efficiency Management Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to trigger an optimization cycle (buildingId + zone), live list of cycles with status pills, expand-row to see HVAC / lighting / equipment reports + consolidated summary + eval score.

Browser title: `<title>Akka Sample: Energy Efficiency Management Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `HvacSpecialist`, `LightingSpecialist`, `EquipmentSpecialist`): checks every proposed control action against safety thresholds and occupant-comfort constraints before the action executes. Blocking. Rejected actions are logged as `blockedByGuardrail = true` in their subsystem report and excluded from physical execution.

## 9. Agent prompts

- `EnergyCoordinator` → `prompts/energy-coordinator.md`. Decomposes the cycle into subsystem queries; later consolidates three specialist reports into the final efficiency report.
- `HvacSpecialist` → `prompts/hvac-specialist.md`. Analyzes HVAC telemetry; returns `HvacReport`.
- `LightingSpecialist` → `prompts/lighting-specialist.md`. Analyzes occupancy and daylight data; returns `LightingReport`.
- `EquipmentSpecialist` → `prompts/equipment-specialist.md`. Identifies inefficient equipment; returns `EquipmentReport`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Trigger a cycle; cycle progresses QUEUED → DISPATCHED → CONSOLIDATING → COMPLETE within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a specialist timeout (set `HvacSpecialist` step timeout to 1 s); cycle enters PARTIAL with whichever specialist reports came back.
3. **J3** — Inject a control action that exceeds the safety threshold; the guardrail rejects it, logs it as `blockedByGuardrail = true`, and the cycle completes without executing that action.
4. **J4** — Wait after a successful cycle; the cycle's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named energy-management-team demonstrating the
delegation-supervisor-workers × ops-automation cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-ops-automation-energy-management-team.
Java package io.akka.samples.energyefficiencymanagementagent. Akka 3.6.0.
HTTP port 9371.

Components to wire (exactly):
- 4 AutonomousAgents:
  * EnergyCoordinator — definition() with capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/energy-coordinator.md. Returns OptimizationPlan{hvacQuery, lightingQuery,
    equipmentQuery} for PLAN and ConsolidatedReport{summary, hvacRecommendations,
    lightingRecommendations, equipmentRecommendations, estimatedSavingsKwh, guardrailVerdict,
    consolidatedAt} for CONSOLIDATE.
  * HvacSpecialist — capability(TaskAcceptance.of(ANALYSE_HVAC).maxIterationsPerTask(3)).
    System prompt from prompts/hvac-specialist.md. Returns HvacReport{recommendations:
    List<SetpointRecommendation{unit, parameter, currentValue, recommendedValue, rationale}>,
    currentLoadKw, analysedAt}. Before-tool-call guardrail on every control action call.
  * LightingSpecialist — capability(TaskAcceptance.of(ANALYSE_LIGHTING).maxIterationsPerTask(3)).
    System prompt from prompts/lighting-specialist.md. Returns LightingReport{scheduleChanges:
    List<LightingScheduleEntry{zone, startTime, endTime, targetLuxLevel, rationale}>,
    currentLoadKw, analysedAt}. Before-tool-call guardrail on every control action call.
  * EquipmentSpecialist — capability(TaskAcceptance.of(ANALYSE_EQUIPMENT).maxIterationsPerTask(3)).
    System prompt from prompts/equipment-specialist.md. Returns EquipmentReport{actions:
    List<EquipmentAction{assetId, actionType, rationale, blockedByGuardrail}>,
    currentLoadKw, analysedAt}. Before-tool-call guardrail on every control action call.

- 1 Workflow OptimizationWorkflow with steps:
  planStep -> [parallel] hvacStep, lightingStep, equipmentStep -> joinStep -> consolidateStep -> emitStep.
  planStep calls forAutonomousAgent(EnergyCoordinator.class, PLAN).
  hvacStep, lightingStep, equipmentStep run in parallel (CompletionStage zip of all three);
  each governed by WorkflowSettings.builder().stepTimeout(OptimizationWorkflow::hvacStep, ofSeconds(90))
  and equivalent for lightingStep and equipmentStep. On any timeout, transition to a partialStep
  that calls consolidateStep with whichever reports arrived, then ends with CyclePartial.
  consolidateStep calls forAutonomousAgent(EnergyCoordinator.class, CONSOLIDATE) with all three
  reports; give consolidateStep a 120s stepTimeout. WorkflowSettings is nested inside Workflow —
  no import.

- 1 EventSourcedEntity OptimizationCycleEntity holding state OptimizationCycle{cycleId,
  buildingId, zone, CycleStatus, Optional<HvacReport> hvacReport,
  Optional<LightingReport> lightingReport, Optional<EquipmentReport> equipmentReport,
  Optional<ConsolidatedReport> consolidated, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant startedAt,
  Optional<Instant> finishedAt}. CycleStatus enum: QUEUED, DISPATCHED, CONSOLIDATING,
  COMPLETE, PARTIAL, BLOCKED. Events: CycleStarted, HvacReportAttached, LightingReportAttached,
  EquipmentReportAttached, CycleConsolidated, CyclePartial, CycleBlocked, EvalScored.
  Commands: startCycle, attachHvacReport, attachLightingReport, attachEquipmentReport,
  consolidate, markPartial, block, recordEval, getCycle.
  emptyState() returns OptimizationCycle.initial("", "", "") with no commandContext() reference.

- 1 EventSourcedEntity TelemetryQueue with command receiveTelemetry(buildingId, zone)
  emitting TelemetryReceived{cycleId, buildingId, zone, recordedAt}.

- 1 View CycleView with row type OptimizationCycleRow (mirrors OptimizationCycle minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  OptimizationCycleEntity events. ONE query getAllCycles SELECT * AS cycles FROM cycle_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TelemetryConsumer subscribed to TelemetryQueue events; on TelemetryReceived
  starts an OptimizationWorkflow with the cycleId as the workflow id.

- 2 TimedActions:
  * TelemetrySimulator — every 90s, reads next line from
    src/main/resources/sample-events/building-telemetry.jsonl and calls
    TelemetryQueue.receiveTelemetry.
  * EfficiencyEvalSampler — every 10 minutes, queries CycleView.getAllCycles, picks the oldest
    COMPLETE cycle without an evalScore, runs a 1–5 rubric judge over the consolidated report,
    then calls OptimizationCycleEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * CycleEndpoint at /api with POST /cycles, GET /cycles, GET /cycles/{id},
    GET /cycles/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- EnergyTasks.java declaring five Task<R> constants: PLAN (OptimizationPlan), ANALYSE_HVAC
  (HvacReport), ANALYSE_LIGHTING (LightingReport), ANALYSE_EQUIPMENT (EquipmentReport),
  CONSOLIDATE (ConsolidatedReport).
- Domain records OptimizationPlan, SetpointRecommendation, HvacReport, LightingScheduleEntry,
  LightingReport, EquipmentAction, EquipmentReport, ConsolidatedReport.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9371 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/building-telemetry.jsonl with 8 canned telemetry lines
  covering different zones (HVAC-A, LIGHTING-B, EQUIP-C, etc.).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control (G1 before-tool-call guardrail) and
  a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = ops-automation,
  decisions.authority_level = recommend-only, data.pii = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/energy-coordinator.md, prompts/hvac-specialist.md, prompts/lighting-specialist.md,
  prompts/equipment-specialist.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Energy Efficiency Management Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Energy Efficiency Management Agent</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (energy-coordinator.json,
  hvac-specialist.json, lighting-specialist.json, equipment-specialist.json),
  picks one entry pseudo-randomly per call, and deserialises it into the
  agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    energy-coordinator.json — list of either OptimizationPlan or ConsolidatedReport objects.
      4–6 OptimizationPlan entries (hvacQuery + lightingQuery + equipmentQuery triples) and
      4–6 ConsolidatedReport entries (each with a 80–120 word summary, representative
      hvacRecommendations, lightingRecommendations, equipmentRecommendations objects,
      estimatedSavingsKwh between 5.0 and 40.0, guardrailVerdict = "ok").
    hvac-specialist.json — 4–6 HvacReport entries, each with 2–4 SetpointRecommendation
      objects naming real HVAC parameters (cooling setpoint, supply air temperature,
      chiller staging threshold).
    lighting-specialist.json — 4–6 LightingReport entries, each with 2–4 LightingScheduleEntry
      objects covering realistic zones and lux targets.
    equipment-specialist.json — 4–6 EquipmentReport entries, each with 2–4 EquipmentAction
      objects with actionType in [LOAD_SHIFT, SETBACK, SHUTDOWN_STANDBY],
      blockedByGuardrail = false.
- A MockModelProvider.seedFor(cycleId) helper makes the selection deterministic per cycle id
  so the same cycle produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (90s for
  specialist steps, 120s for consolidation); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion EnergyTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9371 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip across all three specialist steps, NOT
  sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
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
