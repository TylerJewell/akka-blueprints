# SPEC — maps-trip-planner

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Maps Trip Planner.
**One-line pitch:** A user submits a natural-language travel request; one AI agent calls Google Maps MCP tools to resolve real places and travel times, and returns a structured day-by-day itinerary — each day listing stops, transit legs, estimated durations, and a short narrative.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the planning-travel domain. One `TripPlannerAgent` (AutonomousAgent) carries the entire planning decision; the surrounding components only screen the input and audit the output. Three governance mechanisms are wired around the agent:

- A **destination screener** runs inside a Consumer between the raw trip submission and the agent call — so the model never plans a trip to a restricted or sanctioned destination.
- A **before-agent-response guardrail** validates the agent's itinerary on every turn: well-formed JSON, every day references at least one geocoded place, every transit leg has a duration estimate, and the itinerary's date span matches the request. A malformed itinerary triggers a retry inside the same task.
- An **on-decision-eval** runs immediately after each `ItineraryRecorded` event, scoring the itinerary on coverage and realism (does every requested preference appear in at least one day? are durations plausible? is the day count correct?).

The blueprint shows that the single-agent pattern is not ungoverned — three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills a **Trip request** form: origin city, destination(s), travel dates, party size, and a free-text preferences field (e.g., "vegetarian meals, avoid museums, budget mid-range").
2. The user clicks **Plan my trip**. The UI POSTs to `/api/trips` and receives a `tripId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SCREENED` — the screener's pass/block decision is visible in the card detail.
4. If screened BLOCKED, the card shows the block reason (restricted destination or incomplete request); no agent call is made.
5. If screened PASS, within ~10–30 s the workflow's `planStep` completes. The card transitions to `PLANNING` then `ITINERARY_RECORDED`. The itinerary appears: a day-by-day timeline with stop cards (place name, geocoded coordinates, estimated visit duration, short description), transit legs (mode, estimated time), and a preferences coverage summary.
6. Within ~1 s of the itinerary, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale.
7. The user can submit another request; the live list keeps the history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlanningEndpoint` | `HttpEndpoint` | `/api/trips/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TripRequestEntity`, `ItineraryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TripRequestEntity` | `EventSourcedEntity` | Per-trip lifecycle: submitted → screened → planning → itinerary → evaluated. Source of truth. | `PlanningEndpoint`, `RequestScreener`, `PlanningWorkflow` | `ItineraryView` |
| `RequestScreener` | `Consumer` | Subscribes to `TripRequestSubmitted` events; checks destination against blocked list; emits `RequestScreened` or `RequestBlocked` back to the entity. | `TripRequestEntity` events | `TripRequestEntity` |
| `PlanningWorkflow` | `Workflow` | One workflow per trip. Steps: `awaitScreenedStep` → `planStep` → `evalStep`. | started by `RequestScreener` once screened event lands | `TripPlannerAgent`, `TripRequestEntity` |
| `TripPlannerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives trip request details in the task definition; calls Maps MCP tools (geocode, directions, place-details) to resolve places and transit; returns `TripItinerary`. | invoked by `PlanningWorkflow` | returns itinerary |
| `ItineraryView` | `View` | Read model: one row per trip for the UI. | `TripRequestEntity` events | `PlanningEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TripRequest(
    String tripId,
    String originCity,
    List<String> destinations,
    LocalDate startDate,
    LocalDate endDate,
    int partySize,
    String preferencesText,
    String requestedBy,
    Instant submittedAt
) {}

record ScreenResult(
    ScreenDecision decision,
    String reason
) {}
enum ScreenDecision { PASS, BLOCKED }

record PlaceStop(
    String placeName,
    String geocodedAddress,
    double latDeg,
    double lngDeg,
    int visitMinutes,
    String description
) {}

record TransitLeg(
    String fromPlace,
    String toPlace,
    TransitMode mode,
    int durationMinutes
) {}
enum TransitMode { WALK, DRIVE, TRANSIT, CYCLE }

record ItineraryDay(
    int dayNumber,
    LocalDate date,
    List<PlaceStop> stops,
    List<TransitLeg> legs,
    String dayNarrative
) {}

record TripItinerary(
    ItineraryQuality quality,
    String summary,
    List<ItineraryDay> days,
    List<String> preferencesAddressed,
    Instant decidedAt
) {}
enum ItineraryQuality { EXCELLENT, GOOD, PARTIAL }

record CoverageEval(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record TripPlan(
    String tripId,
    Optional<TripRequest> request,
    Optional<ScreenResult> screen,
    Optional<TripItinerary> itinerary,
    Optional<CoverageEval> eval,
    TripStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TripStatus {
    SUBMITTED, SCREENED, BLOCKED, PLANNING, ITINERARY_RECORDED, EVALUATED, FAILED
}
```

Events on `TripRequestEntity`: `TripRequestSubmitted`, `RequestScreened`, `RequestBlocked`, `PlanningStarted`, `ItineraryRecorded`, `EvaluationScored`, `PlanningFailed`.

Every nullable lifecycle field on the `TripPlan` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/trips` — body `{ originCity, destinations: [String], startDate, endDate, partySize, preferencesText, requestedBy }` → `{ tripId }`.
- `GET /api/trips` — list all trips, newest-first.
- `GET /api/trips/{id}` — one trip.
- `GET /api/trips/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Maps Trip Planner</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted trips (status pill + quality badge + age) and a right pane with the selected trip's detail — request summary, screen result, itinerary day timeline (stops + legs), preferences coverage, and eval score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Destination screener** (`screener`, applied inside `RequestScreener` Consumer): checks every requested destination against a configurable blocked list (sanctioned countries, internally restricted regions). A blocked request never reaches the agent; the entity transitions to BLOCKED and the UI surfaces the reason.
- **G1 — before-agent-response guardrail**: runs on every turn of `TripPlannerAgent`. Asserts the candidate response is well-formed `TripItinerary` JSON, every `days[].stops` list is non-empty, every `TransitLeg.durationMinutes` is positive, and the itinerary day count matches `endDate - startDate` from the request. On failure, returns a structured `invalid-itinerary` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `ItineraryRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that every preference keyword from `preferencesText` appears somewhere in the itinerary narrative or stop descriptions, that no single day exceeds a plausible activity window (16 hours of estimated minutes), and that transit legs connect stops in day order. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `TripPlannerAgent` → `prompts/trip-planner.md`. The single decision-making LLM. System prompt instructs it to use Maps MCP tools to geocode each destination and stop, retrieve directions between stops, and assemble one `ItineraryDay` per requested travel day.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a 3-day Paris trip request; within 30 s the itinerary appears with at least 2 stops per day, transit legs connecting them, and an eval score chip.
2. **J2** — The agent's first response on a trip is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed itinerary; the UI never displays the malformed response.
3. **J3** — A trip request for a destination on the screener's blocked list never reaches the agent; the UI card shows BLOCKED with the reason string within ~1 s of submission.
4. **J4** — A trip itinerary where all transit legs have `durationMinutes = 0` receives an eval score of 1 with a rationale explaining that transit durations are implausible.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named maps-trip-planner demonstrating the single-agent × planning-travel cell.
Runs out of the box (no external services; Maps MCP is simulated in-process).
Maven group io.akka.samples. Maven artifact single-agent-planning-travel-maps-trip-planner.
Java package io.akka.samples.travelplannergooglemapsmcp. Akka 3.6.0. HTTP port 9982.

Components to wire (exactly):

- 1 AutonomousAgent TripPlannerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/trip-planner.md>) and
  .capability(TaskAcceptance.of(PLAN_TRIP).maxIterationsPerTask(3)).
  The task receives the full TripRequest fields as its instruction text.
  Output: TripItinerary{quality: ItineraryQuality (EXCELLENT/GOOD/PARTIAL), summary: String,
  days: List<ItineraryDay>, preferencesAddressed: List<String>, decidedAt: Instant}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries the response within its 3-iteration budget.
  Maps MCP tool calls (geocode, directions, place-details) are declared as tool stubs on the
  agent definition; the in-process MockMapsClient resolves them without a live HTTP call.

- 1 Workflow PlanningWorkflow per tripId with three steps:
  * awaitScreenedStep — polls TripRequestEntity.getTripPlan every 1s; on plan.screen().isPresent()
    AND plan.status() != BLOCKED advances to planStep. If status is BLOCKED, transitions the
    workflow to a terminal no-op (no agent call). WorkflowSettings.stepTimeout 15s.
  * planStep — emits PlanningStarted, then calls componentClient.forAutonomousAgent(
    TripPlannerAgent.class, "planner-" + tripId).runSingleTask(
      TaskDef.instructions(formatTripRequest(plan.request()))
    ) — returns a taskId, then forTask(taskId).result(PLAN_TRIP) to fetch the itinerary.
    On success calls TripRequestEntity.recordItinerary(itinerary). WorkflowSettings.stepTimeout
    90s (Maps tool calls add latency) with defaultStepRecovery maxRetries(2)
    .failoverTo(PlanningWorkflow::error).
  * evalStep — runs a deterministic rule-based CoverageScorer (NOT an LLM call) over the
    recorded itinerary: checks preference coverage, per-day duration sanity, and transit-leg
    ordering. Emits EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings
    .stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TripRequestEntity (one per tripId). State TripPlan{tripId: String,
  request: Optional<TripRequest>, screen: Optional<ScreenResult>,
  itinerary: Optional<TripItinerary>, eval: Optional<CoverageEval>, status: TripStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. TripStatus enum: SUBMITTED, SCREENED,
  BLOCKED, PLANNING, ITINERARY_RECORDED, EVALUATED, FAILED. Events: TripRequestSubmitted{request},
  RequestScreened{screen}, RequestBlocked{reason}, PlanningStarted{}, ItineraryRecorded{itinerary},
  EvaluationScored{eval}, PlanningFailed{reason}. Commands: submit, attachScreen, block,
  markPlanning, recordItinerary, recordEvaluation, fail, getTripPlan. emptyState() returns
  TripPlan.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer RequestScreener subscribed to TripRequestEntity events; on TripRequestSubmitted
  checks every entry in request.destinations() against a blocked-destinations list loaded
  from src/main/resources/blocked-destinations.txt (one destination keyword per line,
  case-insensitive substring match). If any destination matches: calls
  TripRequestEntity.block(reason). If none match: builds ScreenResult(PASS, "ok"), calls
  TripRequestEntity.attachScreen(screen), then starts a PlanningWorkflow with
  id = "plan-" + tripId.

- 1 View ItineraryView with row type TripRow (mirrors TripPlan minus request.preferencesText
  raw form — the view stores a trimmed preferences summary). Table updater consumes
  TripRequestEntity events. ONE query getAllTrips: SELECT * AS trips FROM itinerary_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * PlanningEndpoint at /api with POST /trips (body
    {originCity, destinations: [String], startDate, endDate, partySize, preferencesText,
    requestedBy}; mints tripId; calls TripRequestEntity.submit; returns {tripId}), GET /trips
    (list from getAllTrips, sorted newest-first), GET /trips/{id} (one row), GET
    /trips/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- TripTasks.java declaring one Task<R> constant: PLAN_TRIP = Task.name("Plan trip")
  .description("Use Maps tools to geocode destinations and produce a TripItinerary with one
  ItineraryDay per requested travel date").resultConformsTo(TripItinerary.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TripRequest, ScreenResult, ScreenDecision, PlaceStop, TransitLeg, TransitMode,
  ItineraryDay, TripItinerary, ItineraryQuality, CoverageEval, TripPlan, TripStatus.

- ItineraryGuardrail.java implementing the before-agent-response hook. Reads the candidate
  TripItinerary from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: TripItinerary and the
  original TripRequest. Outputs: CoverageEval. Scoring rubric documented in Javadoc on the class.

- MockMapsClient.java — an in-process stub implementing the Maps MCP tool interface.
  geocode(query) returns a fixed PlaceStop with a realistic lat/lng for well-known city names
  (Paris, Rome, Tokyo, New York, London, Berlin, Barcelona — 7 cities seeded). directions(from,
  to, mode) returns a TransitLeg with a plausible duration based on straight-line distance
  approximation. place-details(placeId) returns a canned description. Unknown queries return a
  sensible default (lat/lng 0/0, duration 60 min). The agent's tool declarations reference
  MockMapsClient methods via the tool-stub wiring in TripPlannerAgent.definition().

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9982 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/trip-scenarios.jsonl with 3 seeded trip requests:
  a 3-day Paris city break (2 adults, cultural + food preferences), a 5-day Amalfi Coast
  road trip (4 adults, outdoor + seafood), and a 2-day Tokyo highlights sprint (1 adult,
  tech + budget preferences).

- src/main/resources/blocked-destinations.txt with 5 seeded blocked destination keywords:
  "restricted-zone-alpha", "embargo-territory", "demo-block-a", "demo-block-b",
  "sanctioned-region-x".

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with controls (S1, G1, E1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for planning-travel domain.

- prompts/trip-planner.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Maps Trip Planner", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of trip cards; right = selected-trip detail with request summary, screen result,
  day-by-day itinerary, and eval-score chip).
  Browser title exactly: <title>Akka Sample: Maps Trip Planner</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(tripId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    plan-trip.json — 8 TripItinerary entries covering the three ItineraryQuality values.
      Each entry has a summary paragraph and a `days` array with ItineraryDays matching the
      seeded Paris, Amalfi, and Tokyo scenarios. Each day has at least 2 PlaceStops with
      geocoded addresses and positive visitMinutes, at least 1 TransitLeg with positive
      durationMinutes, and a dayNarrative paragraph. preferencesAddressed lists the matching
      keywords. Plus 2 deliberately MALFORMED entries (one with a day whose stops list is
      empty; one with a transit leg where durationMinutes = -5) — the guardrail blocks both,
      exercising the retry path. The mock should select a malformed entry on the FIRST
      iteration of every 3rd trip (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(tripId) helper makes per-trip selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TripPlannerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TripTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (planStep
  90s, awaitScreenedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the TripPlan row record is Optional<T>.
- Lesson 7: TripTasks.java with PLAN_TRIP = Task.name(...).description(...)
  .resultConformsTo(TripItinerary.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9982 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (TripPlannerAgent).
  The on-decision eval is rule-based (CoverageScorer.java) and does NOT make an LLM call.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
