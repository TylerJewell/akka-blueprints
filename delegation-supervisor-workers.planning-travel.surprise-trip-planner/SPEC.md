# SPEC — surprise-trip-planner

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 chains the rest of the pipeline.

---

## 1. System name + pitch

**System name:** Surprise Trip Planner.
**One-line pitch:** A traveler enters preferences (vibe, budget band, dates, no-go list); a supervisor agent delegates to three worker agents and returns a surprise itinerary — destination, logistics, and a day-by-day activities plan — with the destination revealed only at the end.

## 2. What this blueprint demonstrates

The delegation-supervisor-workers coordination pattern: one supervisor decomposes a request into worker subtasks, delegates each to a specialist agent, and assembles the typed worker results into a final itinerary. The governance pattern is two guardrails — a before-tool-call hook that constrains which external search/booking tools a worker may invoke and with what scope, and a before-agent-response hook that grounds the assembled travel facts (visa rules, price bands) before the itinerary reaches the traveler.

## 3. User-facing flows

1. The traveler submits a preference set through the App UI or `POST /api/trips`. The response carries a `tripId`; the trip appears in `REQUESTED`.
2. The `TripWorkflow` runs the supervisor, which delegates to `DestinationAgent`; the trip moves to `RESEARCHED` with a chosen destination (hidden from the traveler until the itinerary is ready).
3. The supervisor delegates to `LogisticsAgent` and `ActivitiesAgent`; their typed results are assembled. The before-agent-response guardrail grounds the facts; the trip moves to `READY` with a full itinerary.
4. If a worker's tool call exceeds its granted scope, the before-tool-call guardrail blocks it and the workflow records the block; the trip continues with the worker's safe fallback.
5. A trip stuck mid-plan past the stall threshold is moved to `ESCALATED` by the monitor and the App UI surfaces it.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| TripSupervisorAgent | AutonomousAgent | Decomposes preferences, delegates to workers, assembles itinerary | TripWorkflow | DestinationAgent, LogisticsAgent, ActivitiesAgent |
| DestinationAgent | AutonomousAgent | Picks a destination matching the vibe/budget/no-go list | TripWorkflow | TripEntity |
| LogisticsAgent | AutonomousAgent | Plans transport + lodging for the chosen destination | TripWorkflow | TripEntity |
| ActivitiesAgent | AutonomousAgent | Builds a day-by-day activities plan | TripWorkflow | TripEntity |
| TripWorkflow | Workflow | Orchestrates delegate → assemble → ground steps | TripRequestConsumer | TripEntity, all agents |
| TripEntity | EventSourcedEntity | Holds the trip lifecycle state | TripWorkflow | TripsView |
| TripsView | View | CQRS read model of trips | TripEntity | TripEndpoint |
| TripRequestConsumer | Consumer | Starts a workflow per inbound request | InboundRequestQueue | TripWorkflow |
| InboundRequestQueue | EventSourcedEntity | Buffers inbound preference requests | RequestSimulator, TripEndpoint | TripRequestConsumer |
| RequestSimulator | TimedAction | Drips canned preference sets every 30s | sample-events JSONL | InboundRequestQueue |
| StalledTripMonitor | TimedAction | Escalates trips stalled past threshold | TripsView | TripEntity |
| TripEndpoint | HttpEndpoint | REST + SSE + metadata | UI, clients | InboundRequestQueue, TripsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | browser | static-resources |

Names matter. `/akka:specify` uses them verbatim.

## 5. Data model

See `reference/data-model.md`. Authoritative records:

- `Trip` — entity state and view row. `id`, `preferences` (PreferenceSet), `status` (TripStatus enum), and Optional lifecycle fields: `requestedAt`, `destinationName`, `destinationRationale`, `researchedAt`, `logistics` (Logistics), `activities` (List or null), `assembledAt`, `groundingNotes`, `blockedToolNote`, `escalatedAt`, `failureReason`.
- `PreferenceSet` — `vibe`, `budgetBand`, `startDate`, `endDate`, `partySize`, `noGo` (List).
- `TripStatus` enum — `REQUESTED`, `RESEARCHED`, `READY`, `ESCALATED`, `FAILED`.
- Events: `TripRequested`, `DestinationChosen`, `LogisticsPlanned`, `ActivitiesPlanned`, `ItineraryAssembled`, `ToolCallBlocked`, `TripEscalated`, `TripFailed`.

Every nullable lifecycle field is `Optional<T>` on the row record (Lesson 6).

## 6. API contract

See `reference/api-contract.md` for payload schemas and SSE format. Top-level surface:

```
POST /api/trips                       -> { tripId }
GET  /api/trips ?status=...           -> { trips: [Trip, ...] }
GET  /api/trips/{tripId}              -> Trip
GET  /api/trips/sse                   -> Server-Sent Events of Trip
GET  /api/metadata/eval-matrix        -> text/yaml
GET  /api/metadata/risk-survey        -> text/yaml
GET  /api/metadata/readme             -> text/markdown
GET  /                                -> 302 /app/index.html
GET  /app/{*path}                     -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no `ui/` folder, no npm build. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Browser title: `<title>Akka Sample: Surprise Trip Planner</title>`.

- Overview renders eyebrow "Overview" + headline (sample type, no subtitle) + four cards (Try it / How it works / Components / API contract).
- Architecture renders the four mermaid diagrams from `PLAN.md` with the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- Risk Survey and Eval Matrix render in the `matrix-card` / `matrix-row` style; Eval Matrix labels carry a colored mechanism pill.
- App UI submits a preference set, streams trips via SSE, and reveals the destination only when status is `READY`.

Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index; no hidden zombie panels (Lesson 26). See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two guardrails the generated system wires:

- **G1 (before-tool-call):** before any worker invokes a simulated external search or booking tool, a guardrail checks the call is within the granted scope for that worker; out-of-scope calls are blocked and recorded as a `ToolCallBlocked` event.
- **G2 (before-agent-response):** before the supervisor's assembled itinerary is persisted, a guardrail grounds the travel facts (visa requirements, price bands) against the canned reference data; ungrounded claims are stripped or flagged in `groundingNotes`.

## 9. Agent prompts

- `prompts/trip-supervisor-agent.md` — decomposes preferences and assembles worker results into one itinerary.
- `prompts/destination-agent.md` — selects a destination matching vibe/budget/no-go list.
- `prompts/logistics-agent.md` — plans transport and lodging.
- `prompts/activities-agent.md` — builds the day-by-day activities plan.

## 10. Acceptance

See `reference/user-journeys.md`. Passing journeys:

1. Submit a preference set; within ~30s the trip reaches `READY` with a destination, logistics, and a non-empty activities list.
2. A worker tool call outside its granted scope is blocked by G1; a `ToolCallBlocked` note appears and the trip still completes.
3. An itinerary with an ungrounded travel fact is corrected by G2; `groundingNotes` records what was adjusted.
4. A trip stalled past the threshold transitions to `ESCALATED` without UI interaction.

---

## 11. Implementation directives

```
Create a sample named surprise-trip-planner demonstrating the
delegation-supervisor-workers x planning-travel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
surprise-trip-planner. Java package io.akka.samples.surprisetripplanner.
Akka 3.6.0. HTTP port 9525.

Components to wire (exactly):
- 4 AutonomousAgents:
  - TripSupervisorAgent: definition() with instructions to decompose a
    PreferenceSet and assemble worker outputs into an Itinerary; capability
    (TaskAcceptance.of(ASSEMBLE).maxIterationsPerTask(3)). Returns typed
    Itinerary{destinationName,destinationRationale,logistics,activities,
    groundingNotes}.
  - DestinationAgent: returns typed Destination{name,rationale}. capability
    (TaskAcceptance.of(CHOOSE_DESTINATION).maxIterationsPerTask(2)).
  - LogisticsAgent: returns typed Logistics{transport,lodging,estCostBand}.
    capability(TaskAcceptance.of(PLAN_LOGISTICS).maxIterationsPerTask(2)).
  - ActivitiesAgent: returns typed Activities{days:List<DayPlan>}. capability
    (TaskAcceptance.of(PLAN_ACTIVITIES).maxIterationsPerTask(2)).
  Each agent has a before-tool-call guardrail (G1) and the supervisor also has
  a before-agent-response guardrail (G2). Never silently downgrade
  AutonomousAgent to Agent (Lesson 1).
- 1 Workflow TripWorkflow with steps: chooseDestinationStep ->
  planLogisticsStep -> planActivitiesStep -> assembleStep. Each agent-calling
  step uses forAutonomousAgent(Agent.class, instanceId).runSingleTask(...)
  then forTask(taskId).result(...). assembleStep calls TripSupervisorAgent and
  then TripEntity.recordItinerary. Override settings() with stepTimeout(60s) on
  every agent-calling step and defaultStepRecovery(maxRetries(2).failoverTo
  (TripWorkflow::error)). WorkflowSettings is nested inside Workflow — no import
  (Lesson 5).
- 1 EventSourcedEntity TripEntity holding a Trip record: id, preferences
  (PreferenceSet), TripStatus enum, and Optional lifecycle fields requestedAt,
  destinationName, destinationRationale, researchedAt, logistics, activities,
  assembledAt, groundingNotes, blockedToolNote, escalatedAt, failureReason.
  Events: TripRequested, DestinationChosen, LogisticsPlanned, ActivitiesPlanned,
  ItineraryAssembled, ToolCallBlocked, TripEscalated, TripFailed. Commands:
  recordRequest, recordDestination, recordLogistics, recordActivities,
  recordItinerary, recordToolBlock, markEscalated, markFailed, getTrip.
  emptyState() returns Trip.initial("") with NO commandContext() reference
  (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue: single command enqueueRequest
  (PreferenceSet) emitting InboundRequestQueued.
- 1 View TripsView with row type Trip, table updater consuming TripEntity
  events. ONE query: getAllTrips SELECT * AS trips FROM trips_view. No WHERE
  status filter (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 1 Consumer TripRequestConsumer subscribed to InboundRequestQueue events; on
  each event starts a TripWorkflow with a fresh UUID.
- 2 TimedActions: RequestSimulator (every 30s, reads next line from
  src/main/resources/sample-events/trip-requests.jsonl, calls
  InboundRequestQueue.enqueueRequest); StalledTripMonitor (every 30s, queries
  TripsView.getAllTrips, filters trips in REQUESTED/RESEARCHED with requestedAt
  older than 3 minutes, calls TripEntity.markEscalated).
- 2 HttpEndpoints: TripEndpoint at /api (create trip, list trips filtered
  client-side, single trip, SSE stream, three /api/metadata/* endpoints serving
  files from src/main/resources/metadata/); AppEndpoint serving / -> 302
  /app/index.html and /app/* -> static-resources/*.

Companion files:
- SupervisorTasks.java declaring Task<R> constants: CHOOSE_DESTINATION
  (resultConformsTo Destination), PLAN_LOGISTICS (Logistics), PLAN_ACTIVITIES
  (Activities), ASSEMBLE (Itinerary). Lesson 7 — AutonomousAgent requires this
  companion or it is a compile error.
- Records: PreferenceSet(String vibe, String budgetBand, String startDate,
  String endDate, int partySize, List<String> noGo), Destination(String name,
  String rationale), Logistics(String transport, String lodging, String
  estCostBand), DayPlan(int day, String summary, List<String> items),
  Activities(List<DayPlan> days), Itinerary(String destinationName, String
  destinationRationale, Logistics logistics, List<DayPlan> activities, String
  groundingNotes).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9525 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names current before locking (Lesson 8).
- src/main/resources/sample-events/trip-requests.jsonl with 8 canned preference
  sets.
- src/main/resources/reference-facts/ canned visa + price-band data the G2
  grounding check reads.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the root with the 2 controls (G1, G2) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the root (already authored): elevator pitch, prerequisites,
  generate steps, component inventory, customise, validation, license. NO
  governance-mechanisms section, NO configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — single self-contained file
  (Lesson 17). Inline CSS + JS; runtime CDN imports for marked and js-yaml are
  acceptable. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App
  UI. Match the governance.html visual style (dark / yellow accent / Instrument
  Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (TripSupervisorAgent -> Itinerary, DestinationAgent -> Destination,
  LogisticsAgent -> Logistics, ActivitiesAgent -> Activities; write 4-6 entries
  each to src/main/resources/mock-responses/{trip-supervisor,destination,
  logistics,activities}-agent.json). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the REFERENCE
  — env-var name, file path, or secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes captured key
  material.

Mock LLM provider — per-agent mock-response shapes:
- trip-supervisor-agent.json: Itinerary objects with a destinationName, a
  one-line rationale, a Logistics block, a 3-5 entry day list, and a
  groundingNotes string.
- destination-agent.json: Destination objects {name, rationale} for a range of
  vibes (coastal, alpine, city-culture, desert).
- logistics-agent.json: Logistics objects {transport, lodging, estCostBand}.
- activities-agent.json: Activities objects with a 3-5 entry days list.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: every agent-calling workflow step sets stepTimeout(60s).
- Lesson 6: Optional<T> for every nullable lifecycle field on the row record.
- Lesson 7: AutonomousAgent requires the SupervisorTasks.java companion.
- Lesson 8: verify model names current before locking.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit http-port 9525 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column, no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box", never tier codes.
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: mermaid state-label CSS overrides + edge-label overflow:visible +
  transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key after the Lesson 25 sourcing flow, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
