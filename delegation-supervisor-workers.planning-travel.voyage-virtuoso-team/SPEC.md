# SPEC — voyage-virtuoso-multi-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Voyage Virtuoso Multi-Agent.
**One-line pitch:** Submit a travel request; a director delegates flight, accommodation, experience, and logistics work to four specialists in parallel, then assembles one premium itinerary.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to four AutonomousAgents in parallel, gathers their results, and asks a fifth AutonomousAgent to assemble a unified itinerary. The blueprint also demonstrates an **output guardrail** that vets the customer-facing itinerary before it is returned, and an **eval-event** that samples the director's assembly decision for quality scoring.

## 3. User-facing flows

The user opens the App UI tab and submits a travel request via the form.

1. The system creates a `TravelRequest` record in `PLANNING` and starts an `ItineraryWorkflow`.
2. The ItineraryDirector decomposes the request into four parallel work items: a routing brief for FlightSpecialist, a lodging brief for AccommodationSpecialist, an activities brief for ExperienceSpecialist, and an entry-requirements brief for LogisticsSpecialist.
3. The workflow forks: all four specialists run concurrently. Each returns a typed payload.
4. The ItineraryDirector assembles the four payloads into an `ItineraryPlan { synopsis, flightPlan, lodgingPlan, experiencePlan, logisticsPlan, guardrailVerdict }`.
5. An output guardrail vets the assembled itinerary for fabricated fares, non-existent properties, or policy violations; if it fails, the request moves to `BLOCKED`. Otherwise the request moves to `ASSEMBLED`.
6. If any specialist times out after 60 seconds, the workflow short-circuits: it asks the ItineraryDirector to assemble from whichever specialists returned, and the request enters `PARTIAL`.

A `RequestSimulator` (TimedAction) drips a sample travel scenario every 60 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ItineraryDirector` | `AutonomousAgent` | Decomposes the travel request, assembles the final itinerary, runs the output guardrail. | `ItineraryWorkflow` | returns typed result to workflow |
| `FlightSpecialist` | `AutonomousAgent` | Identifies routing options and fare classes. Seeded "GDS tool" returns canned results. | `ItineraryWorkflow` | — |
| `AccommodationSpecialist` | `AutonomousAgent` | Selects lodging properties matching tier and preferences. | `ItineraryWorkflow` | — |
| `ExperienceSpecialist` | `AutonomousAgent` | Curates activities, dining, and cultural highlights. | `ItineraryWorkflow` | — |
| `LogisticsSpecialist` | `AutonomousAgent` | Plans ground transport, transfers, and entry requirements. | `ItineraryWorkflow` | — |
| `ItineraryWorkflow` | `Workflow` | Coordinates parallel fan-out to all four specialists, the assembly call, and the guardrail. | `TravelEndpoint`, `TravelRequestConsumer` | `TravelRequestEntity` |
| `TravelRequestEntity` | `EventSourcedEntity` | Holds the request's lifecycle (planning → in-progress → assembled / partial / blocked). | `ItineraryWorkflow` | `TravelRequestView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted travel request for replay and audit. | `TravelEndpoint`, `RequestSimulator` | `TravelRequestConsumer` |
| `TravelRequestView` | `View` | List-of-requests read model. | `TravelRequestEntity` events | `TravelEndpoint` |
| `TravelRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `ItineraryWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample travel scenario every 60 s. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one assembled itinerary every 5 minutes for eval scoring; emits an `EvalScored` event. | scheduler | `TravelRequestEntity` |
| `TravelEndpoint` | `HttpEndpoint` | `/api/travel/*` — submit, get, list, SSE. | — | `TravelRequestView`, `RequestQueue`, `TravelRequestEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TravelQuery(String destination, String origin, String departureDate,
                   String returnDate, int travellers, String tier, String requestedBy) {}

record FlightPlan(String outboundRouting, String returnRouting, String fareClass,
                  String cabinClass, List<String> notes, Instant gatheredAt) {}

record LodgingPlan(String propertyName, String propertyCategory, String roomType,
                   List<String> highlights, Instant gatheredAt) {}

record ExperiencePlan(List<String> activities, List<String> dining,
                      List<String> cultural, Instant gatheredAt) {}

record LogisticsPlan(List<String> groundTransfers, List<String> entryRequirements,
                     List<String> travelAdvisories, Instant gatheredAt) {}

record AssemblyBrief(FlightPlan flight, LodgingPlan lodging,
                     ExperiencePlan experience, LogisticsPlan logistics) {}

record ItineraryPlan(String synopsis, FlightPlan flightPlan, LodgingPlan lodgingPlan,
                     ExperiencePlan experiencePlan, LogisticsPlan logisticsPlan,
                     String guardrailVerdict, Instant assembledAt) {}

record TravelRequest(
    String requestId,
    String destination,
    String origin,
    String tier,
    RequestStatus status,
    Optional<FlightPlan> flightPlan,
    Optional<LodgingPlan> lodgingPlan,
    Optional<ExperiencePlan> experiencePlan,
    Optional<LogisticsPlan> logisticsPlan,
    Optional<ItineraryPlan> itinerary,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus { PLANNING, IN_PROGRESS, ASSEMBLED, PARTIAL, BLOCKED }
```

### Events (on `TravelRequestEntity`)

`RequestCreated`, `FlightPlanAttached`, `LodgingPlanAttached`, `ExperiencePlanAttached`,
`LogisticsPlanAttached`, `ItineraryAssembled`, `ItineraryPartial`, `RequestBlocked`, `EvalScored`.

### Events (on `RequestQueue`)

`TravelSubmitted { requestId, destination, origin, tier, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/travel` — body `{ destination, origin, departureDate, returnDate, travellers, tier }` → `{ requestId }`. Starts a workflow.
- `GET /api/travel` — list all requests. Optional `?status=PLANNING|IN_PROGRESS|ASSEMBLED|PARTIAL|BLOCKED`.
- `GET /api/travel/{id}` — one request.
- `GET /api/travel/sse` — server-sent events stream of every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Voyage Virtuoso Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a travel request, live list of requests with status pills, expand-row to see flight plan, lodging, experiences, logistics, assembled synopsis, and eval score.

Browser title: `<title>Akka Sample: Voyage Virtuoso Multi-Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ItineraryDirector`): vets the assembled itinerary for fabricated fares, non-existent properties, and policy violations. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one assembled itinerary every 5 minutes and emits an `EvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `ItineraryDirector` → `prompts/itinerary-director.md`. Decomposes the travel request into pillar-specific briefs; later assembles the four specialist outputs into the final itinerary.
- `FlightSpecialist` → `prompts/flight-specialist.md`. Identifies routing and fare options; returns `FlightPlan`.
- `AccommodationSpecialist` → `prompts/accommodation-specialist.md`. Selects lodging; returns `LodgingPlan`.
- `ExperienceSpecialist` → `prompts/experience-specialist.md`. Curates activities and dining; returns `ExperiencePlan`.
- `LogisticsSpecialist` → `prompts/logistics-specialist.md`. Plans transfers and entry requirements; returns `LogisticsPlan`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a travel request; it progresses PLANNING → IN_PROGRESS → ASSEMBLED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a specialist timeout (set `FlightSpecialist` timeout to 1 s); request enters PARTIAL with whichever pillar outputs arrived.
3. **J3** — Inject a guardrail failure (ItineraryDirector returns a fabricated fare); request enters BLOCKED.
4. **J4** — Wait after a successful assembly; the request row shows an eval score.
5. **J5** — Simulator drips a scenario; the App UI is non-empty on first load without any user submission.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named voyage-virtuoso-multi-agent demonstrating the
delegation-supervisor-workers × planning-travel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-planning-travel-voyage-virtuoso-team.
Java package io.akka.samples.voyagevirtuosomultiagent. Akka 3.6.0. HTTP port 9798.

Components to wire (exactly):
- 5 AutonomousAgents:
  * ItineraryDirector — definition() with
      capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
      AND capability(TaskAcceptance.of(ASSEMBLE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/itinerary-director.md. Returns
    AssemblyBrief{flight,lodging,experience,logistics} for DECOMPOSE and
    ItineraryPlan{synopsis,flightPlan,lodgingPlan,experiencePlan,logisticsPlan,
    guardrailVerdict,assembledAt} for ASSEMBLE.
  * FlightSpecialist — capability(TaskAcceptance.of(PLAN_FLIGHT).maxIterationsPerTask(3)).
    System prompt from prompts/flight-specialist.md. Returns
    FlightPlan{outboundRouting,returnRouting,fareClass,cabinClass,notes,gatheredAt}.
  * AccommodationSpecialist —
    capability(TaskAcceptance.of(PLAN_LODGING).maxIterationsPerTask(3)).
    System prompt from prompts/accommodation-specialist.md. Returns
    LodgingPlan{propertyName,propertyCategory,roomType,highlights,gatheredAt}.
  * ExperienceSpecialist —
    capability(TaskAcceptance.of(PLAN_EXPERIENCE).maxIterationsPerTask(3)).
    System prompt from prompts/experience-specialist.md. Returns
    ExperiencePlan{activities,dining,cultural,gatheredAt}.
  * LogisticsSpecialist —
    capability(TaskAcceptance.of(PLAN_LOGISTICS).maxIterationsPerTask(3)).
    System prompt from prompts/logistics-specialist.md. Returns
    LogisticsPlan{groundTransfers,entryRequirements,travelAdvisories,gatheredAt}.

- 1 Workflow ItineraryWorkflow with steps:
  decomposeStep -> [parallel] flightStep, lodgingStep, experienceStep, logisticsStep
    -> joinStep -> assembleStep -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(ItineraryDirector.class, DECOMPOSE).
  flightStep, lodgingStep, experienceStep, and logisticsStep run in parallel
  (CompletionStage allOf / combine chain); each governed by
  WorkflowSettings.builder().stepTimeout per step at 60s.
  On any specialist timeout, transition to partialStep that calls assembleStep with
  whichever pillar outputs returned, then ends with ItineraryPartial.
  assembleStep calls forAutonomousAgent(ItineraryDirector.class, ASSEMBLE) with the
  merged inputs; give assembleStep a 90s stepTimeout. guardrailStep runs the
  deterministic vetter + LLM judge on the assembled itinerary; on failure, end with
  RequestBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TravelRequestEntity holding state TravelRequest{requestId,
  destination, origin, tier, RequestStatus, Optional<FlightPlan> flightPlan,
  Optional<LodgingPlan> lodgingPlan, Optional<ExperiencePlan> experiencePlan,
  Optional<LogisticsPlan> logisticsPlan, Optional<ItineraryPlan> itinerary,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  RequestStatus enum: PLANNING, IN_PROGRESS, ASSEMBLED, PARTIAL, BLOCKED.
  Events: RequestCreated, FlightPlanAttached, LodgingPlanAttached,
  ExperiencePlanAttached, LogisticsPlanAttached, ItineraryAssembled, ItineraryPartial,
  RequestBlocked, EvalScored.
  Commands: createRequest, attachFlightPlan, attachLodgingPlan, attachExperiencePlan,
  attachLogisticsPlan, assemble, partial, block, recordEval, getRequest.
  emptyState() returns TravelRequest.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueTravel(destination, origin,
  tier, requestedBy) emitting TravelSubmitted{requestId, destination, origin, tier,
  requestedBy, submittedAt}.

- 1 View TravelRequestView with row type TravelRequestRow (mirrors TravelRequest minus
  heavy nested payloads; every nullable field is Optional<T>). Table updater consumes
  TravelRequestEntity events. ONE query getAllRequests SELECT * AS requests FROM
  travel_request_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  caller filters client-side.

- 1 Consumer TravelRequestConsumer subscribed to RequestQueue events; on TravelSubmitted
  starts an ItineraryWorkflow with the requestId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/travel-scenarios.jsonl and calls
    RequestQueue.enqueueTravel.
  * EvalSampler — every 5 minutes, queries TravelRequestView.getAllRequests, picks
    the oldest ASSEMBLED request without an evalScore, runs a 1–5 rubric judge over
    the itinerary synopsis and the four pillar plans, then calls
    TravelRequestEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * TravelEndpoint at /api with POST /travel, GET /travel, GET /travel/{id},
    GET /travel/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TravelTasks.java declaring five Task<R> constants: DECOMPOSE (AssemblyBrief),
  PLAN_FLIGHT (FlightPlan), PLAN_LODGING (LodgingPlan), PLAN_EXPERIENCE (ExperiencePlan),
  PLAN_LOGISTICS (LogisticsPlan), ASSEMBLE (ItineraryPlan).
- Domain records TravelQuery, AssemblyBrief, FlightPlan, LodgingPlan, ExperiencePlan,
  LogisticsPlan, ItineraryPlan.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9798 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/travel-scenarios.jsonl with 8 canned travel scenario
  lines covering different destinations, tiers, and party sizes.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view
  list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  travel-itinerary-assembly, decisions.authority_level = recommend-only,
  data.data_classes.pii = true (destination and traveller count), capabilities.* = false
  except content_generation = true; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/itinerary-director.md, prompts/flight-specialist.md,
  prompts/accommodation-specialist.md, prompts/experience-specialist.md,
  prompts/logistics-specialist.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Voyage Virtuoso Multi-Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Voyage Virtuoso Multi-Agent</title>. No subtitle on Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json, picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return
  shape.
- Per-agent mock-response shapes for THIS blueprint:
    itinerary-director.json — list of either AssemblyBrief or ItineraryPlan
      objects. 4–6 AssemblyBrief entries (each containing placeholder pillar
      sub-objects) and 4–6 ItineraryPlan entries (each with a 100–160 word
      synopsis, a realistic FlightPlan, LodgingPlan, ExperiencePlan,
      LogisticsPlan, guardrailVerdict = "ok").
    flight-specialist.json — 4–6 FlightPlan entries with realistic routing
      strings (e.g., "JFK-CDG via AA 100"), fare class codes, cabin class,
      and 2–3 notes per entry.
    accommodation-specialist.json — 4–6 LodgingPlan entries with property
      names, categories (boutique hotel, resort, urban luxury, etc.), room
      types, and 2–4 highlights.
    experience-specialist.json — 4–6 ExperiencePlan entries each with 3–5
      activities, 2–4 dining recommendations, and 2–3 cultural highlights.
    logistics-specialist.json — 4–6 LogisticsPlan entries with 2–3 ground
      transfer options, 2–4 entry requirement notes, and 1–3 travel
      advisories.
- A MockModelProvider.seedFor(requestId) helper makes the selection
  deterministic per request id so the same request produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s
  specialists, 90s assembly); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion TravelTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9798 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList
  index; delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf / combine, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK
  in narrative, T1/T2/T3/T4, deferred, marketing tone.
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
