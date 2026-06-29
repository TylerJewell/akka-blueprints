# SPEC — trip-planner-team

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–12 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Trip Planner Team.
**One-line pitch:** The user submits a destination, a few preferences, and a trip length; a supervisor delegates to a local-expert agent (which gathers city insights from a simulated search-and-scrape surface), an itinerary agent, and a logistics agent, and returns a single grounded travel plan.

## 2. What this blueprint demonstrates

The delegation-supervisor-workers coordination pattern: one workflow plays supervisor and hands sub-tasks to three specialist worker agents in sequence, persisting each worker's result before delegating the next. The governance pattern wires two guardrails — a before-tool-call URL/robots guard on the local-expert's scrape tool, and a before-agent-response grounding check on the itinerary so the user-facing plan only references facts the local-expert actually gathered.

## 3. User-facing flows

1. The user opens the App UI tab, enters a destination, free-text preferences, and a number of days, and submits. The response carries a `tripId` and the trip appears in `GATHERING_INSIGHTS`.
2. The local-expert agent searches and scrapes the in-process travel surface for the destination; the trip moves to `PLANNING_ITINERARY` with `localInsights` populated.
3. The itinerary agent drafts a day-by-day plan from the preferences and the gathered insights; the trip moves to `PLANNING_LOGISTICS` with `itinerary` populated.
4. The logistics agent plans transport, lodging windows, and daily timing; the supervisor assembles `finalPlan` and the trip moves to `COMPLETED`.
5. If the scrape tool is asked to fetch a non-allowlisted URL, the guard blocks it and the agent continues with search-only insights. If the itinerary references a place absent from the insights, the grounding guard blocks the response and the step retries.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TripPlanningWorkflow` | Workflow | Supervisor; delegates to the three worker agents and assembles the final plan | `TripEndpoint`, `TripRequestConsumer` | `LocalExpertAgent`, `ItineraryAgent`, `LogisticsAgent`, `TripEntity` |
| `LocalExpertAgent` | Agent | Gathers city insights using simulated search + scrape tools | `TripPlanningWorkflow` | `TravelSourceEndpoint` (tools) |
| `ItineraryAgent` | Agent | Drafts a day-by-day itinerary from preferences + insights | `TripPlanningWorkflow` | — |
| `LogisticsAgent` | Agent | Plans transport, lodging windows, daily timing | `TripPlanningWorkflow` | — |
| `TripEntity` | EventSourcedEntity | Holds the trip lifecycle and worker outputs | `TripPlanningWorkflow` | `TripsView` |
| `InboundRequestQueue` | EventSourcedEntity | Buffers simulated trip requests | `TripRequestSimulator` | `TripRequestConsumer` |
| `TripsView` | View | Read model over `TripEntity` events for list + SSE | `TripEntity` | `TripEndpoint` |
| `TripRequestConsumer` | Consumer | Starts a workflow per queued request | `InboundRequestQueue` | `TripPlanningWorkflow` |
| `TripRequestSimulator` | TimedAction | Drips canned trip requests every 30s | `sample-events/trip-requests.jsonl` | `InboundRequestQueue` |
| `TripEndpoint` | HttpEndpoint | `/api` surface: submit, list, single, SSE, metadata | UI / clients | `TripPlanningWorkflow`, `TripsView` |
| `TravelSourceEndpoint` | HttpEndpoint | In-process simulated search + scrape surface | `LocalExpertAgent` tools | `sample-data/*` |
| `AppEndpoint` | HttpEndpoint | Serves the single-file UI | Browser | `static-resources/` |

## 5. Data model

See `reference/data-model.md`. Authoritative record (every nullable lifecycle field is `Optional<T>` per Lesson 6):

```
TripPlan(
  String id,
  String destination,
  String preferences,
  int days,
  TripStatus status,
  Instant createdAt,
  Optional<String> localInsights,
  Optional<Instant> insightsAt,
  Optional<String> itinerary,
  Optional<Instant> itineraryAt,
  Optional<String> logistics,
  Optional<Instant> logisticsAt,
  Optional<String> finalPlan,
  Optional<Instant> completedAt,
  Optional<String> failureReason,
  Optional<Instant> failedAt
)
```

`TripStatus` enum: `GATHERING_INSIGHTS | PLANNING_ITINERARY | PLANNING_LOGISTICS | COMPLETED | FAILED`.

Events: `TripRequested`, `InsightsGathered`, `ItineraryDrafted`, `LogisticsPlanned`, `TripCompleted`, `TripFailed`. `InboundRequestQueue` emits `TripRequestQueued`.

## 6. API contract

See `reference/api-contract.md` for payload schemas and SSE format. Top-level surface:

```
POST /api/trips                   -> { tripId }
GET  /api/trips                   -> { trips: [TripPlan, ...] }
GET  /api/trips/{tripId}          -> TripPlan
GET  /api/trips/sse               -> Server-Sent Events of TripPlan
GET  /api/sim/search?q=...        -> simulated search results (in-process)
GET  /api/sim/scrape?url=...      -> simulated page content (in-process)
GET  /api/metadata/eval-matrix    -> text/yaml
GET  /api/metadata/risk-survey    -> text/yaml
GET  /api/metadata/readme         -> text/markdown
GET  /                            -> 302 /app/index.html
GET  /app/{*path}                 -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm). See `reference/ui-mockup.md`. Browser title: `<title>Akka Sample: Trip Planner Team</title>`. Five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI. Tab switching is attribute-based (`data-tab` / `data-panel`) per Lesson 26, with no hidden zombie panels. The Architecture tab renders the `PLAN.md` mermaid diagrams with the Lesson 24 state-label CSS overrides. The App UI tab submits a trip and streams the live trip list over SSE, showing the four sections (insights, itinerary, logistics, final plan) as they fill in.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two mechanisms the generated system wires:

- **G1 — before-tool-call guard.** Before `LocalExpertAgent` invokes the scrape tool, a guardrail checks the target URL against an allowlist and a robots rule; non-allowlisted or disallowed URLs are blocked and the agent falls back to search-only.
- **G2 — before-agent-response grounding check.** Before `ItineraryAgent` returns, a guardrail verifies every place named in the itinerary appears in the gathered `localInsights`; an ungrounded itinerary is blocked and the step retries.

## 9. Agent prompts

- `prompts/local-expert-agent.md` — gathers destination insights using the search and scrape tools.
- `prompts/itinerary-agent.md` — turns preferences plus insights into a day-by-day plan.
- `prompts/logistics-agent.md` — plans transport, lodging windows, and daily timing.

## 10. Acceptance

See `reference/user-journeys.md`. The journeys whose passing means the blueprint generated correctly:

1. **Plan a trip end to end.** Submit `{destination, preferences, days}`; within ~60s the trip reaches `COMPLETED` with non-empty `localInsights`, `itinerary`, `logistics`, and `finalPlan`.
2. **Scrape guard blocks a bad URL.** The local-expert tool asked to scrape a non-allowlisted URL is blocked before the call; insights are still produced from search.
3. **Grounding guard blocks an ungrounded itinerary.** An itinerary naming a place absent from insights is blocked before reaching the user; the step retries and the trip still completes.
4. **Background load.** With no UI interaction, `TripRequestSimulator` seeds a request and a workflow runs to `COMPLETED`.

---

## 11. Implementation directives

```
Create a sample named trip-planner-team demonstrating the
delegation-supervisor-workers x planning-travel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
trip-planner-team. Java package io.akka.samples.tripplannerteam. Akka 3.6.0.
HTTP port 9385.

Components to wire (exactly):
- 1 Workflow TripPlanningWorkflow (supervisor) with steps gatherInsightsStep
  -> planItineraryStep -> planLogisticsStep -> finalizeStep. Each agent step
  calls componentClient.forAgent().inSession(tripId).method(Agent::method)
  .invoke(arg) and writes the result to TripEntity. finalizeStep assembles
  finalPlan from localInsights + itinerary + logistics, calls
  TripEntity.complete, and ends. Override settings() with stepTimeout(60s) on
  gatherInsightsStep, planItineraryStep, planLogisticsStep, and finalizeStep;
  defaultStepRecovery(maxRetries(2).failoverTo(TripPlanningWorkflow::fail)).
- 3 Agents (request/response, extends akka.javasdk.agent.Agent — never
  AutonomousAgent): LocalExpertAgent.gather(insightRequest) returns
  Insights{summary, sources}; ItineraryAgent.draft(itineraryRequest) returns
  Itinerary{days, text}; LogisticsAgent.plan(logisticsRequest) returns
  Logistics{text}. Each defines a user-named method returning Effect<R> via
  effects().systemMessage(prompt).userMessage(input).responseAs(R.class)
  .thenReply(). Do NOT generate a Tasks.java (that companion is only for
  AutonomousAgent).
- LocalExpertAgent declares two function tools: search(query) and scrape(url).
  Both call TravelSourceEndpoint over the in-process loopback. scrape is wrapped
  by a before-tool-call guardrail (G1) that checks url against an allowlist in
  application.conf (travel.allowed-hosts) and a robots rule; blocked calls
  return a "blocked: not allowlisted" string and the agent proceeds search-only.
- ItineraryAgent is wrapped by a before-agent-response guardrail (G2) that
  parses place names from the returned Itinerary and verifies each appears in
  the localInsights passed into the step; an ungrounded itinerary is rejected
  so the step retries.
- 1 EventSourcedEntity TripEntity holding TripPlan (id, destination,
  preferences, days, TripStatus, createdAt, and Optional lifecycle fields
  localInsights, insightsAt, itinerary, itineraryAt, logistics, logisticsAt,
  finalPlan, completedAt, failureReason, failedAt). Commands: request,
  recordInsights, recordItinerary, recordLogistics, complete, fail, getTrip.
  Events: TripRequested, InsightsGathered, ItineraryDrafted, LogisticsPlanned,
  TripCompleted, TripFailed. emptyState() returns TripPlan.initial("","","",0)
  with no commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue with command enqueue(destination,
  preferences, days) emitting TripRequestQueued.
- 1 View TripsView with row type TripPlan, table updater consuming TripEntity
  events. ONE query: getAllTrips SELECT * AS trips FROM trips_view. No WHERE on
  the status enum (Akka cannot auto-index enum columns, Lesson 2) — filter
  client-side in callers.
- 1 Consumer TripRequestConsumer subscribed to InboundRequestQueue events;
  on each event starts a TripPlanningWorkflow with a fresh UUID.
- 1 TimedAction TripRequestSimulator (every 30s) reads the next line of
  src/main/resources/sample-events/trip-requests.jsonl and calls
  InboundRequestQueue.enqueue.
- 2 HttpEndpoints plus the app endpoint: TripEndpoint at /api (submit, list
  filtered client-side from getAllTrips, single, SSE, and the three
  /api/metadata/* endpoints serving from src/main/resources/metadata/);
  TravelSourceEndpoint at /api/sim (search, scrape) returning canned JSON from
  src/main/resources/sample-data/; AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- Records: Insights(String summary, List<String> sources), Itinerary(int days,
  String text), Logistics(String text). Request records as needed.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9385; travel.allowed-hosts list; akka.javasdk.agent model-provider blocks for
  anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/trip-requests.jsonl with 6 canned requests.
- src/main/resources/sample-data/ with canned search results and page bodies
  for a few destinations (e.g. lisbon.json, kyoto.json).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies the endpoint serves from classpath).
- eval-matrix.yaml at the project root with controls G1, G2 and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per the structure in Section 11 of the authoring
  guide. No governance-mechanisms section, no configuration section.
- src/main/resources/static-resources/index.html — one self-contained file (no
  ui/, no npm). Inline CSS + JS; runtime CDN imports for markdown and YAML are
  fine. Five tabs (Overview, Architecture, Risk Survey, Eval Matrix, App UI).
  Browser title "Akka Sample: Trip Planner Team".

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (LocalExpertAgent -> Insights, ItineraryAgent -> Itinerary,
  LogisticsAgent -> Logistics; see src/main/resources/mock-responses/
  {local-expert,itinerary,logistics}.json with 4-6 entries each). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources it before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes (option a):
- LocalExpertAgent -> Insights{summary: 3-5 sentences on the destination,
  sources: 2-3 allowlisted URLs}.
- ItineraryAgent -> Itinerary{days: requested count, text: a day-by-day plan
  that only names places present in the supplied insights}.
- LogisticsAgent -> Logistics{text: transport + lodging-window + daily-timing
  notes}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: agents extend Agent and are never silently swapped for another
  primitive.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout(60s).
- Lesson 6: Optional<T> on every nullable lifecycle field of the TripPlan row
  record; callers use orElse / isPresent.
- Lesson 7: no Tasks.java since no AutonomousAgent is used.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: the run command is "/akka:build" everywhere, never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9385 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column with no horizontal scroll.
- Lesson 13: descriptive integration label ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label CSS overrides + edge-label foreignObject
  overflow:visible + transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write a key to disk.
- Lesson 26: attribute-based tab switching; no hidden zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9385/`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
