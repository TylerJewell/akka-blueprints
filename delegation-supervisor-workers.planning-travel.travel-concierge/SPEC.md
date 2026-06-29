# SPEC — travel-concierge

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Travel Concierge.
**One-line pitch:** Describe a trip; a coordinator delegates destination research and fare estimation to specialist agents in parallel, then assembles a personalised itinerary and holds for your approval before committing any booking.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to assemble a unified itinerary. The blueprint also demonstrates a **before-tool-call guardrail** that gates every booking action against the user's stated constraints, and a **human-in-the-loop** confirmation step that pauses the workflow at the purchase boundary and requires explicit approval before any booking tool is invoked.

## 3. User-facing flows

The user opens the App UI tab and submits a trip request (destination, dates, budget, traveller count) via the form.

1. The system creates a `Trip` record in `PLANNING` and starts a `TripWorkflow`.
2. The Coordinator decomposes the request into a destination brief for the DestinationScout and a fare query for the FareAgent.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `TripItinerary { destination, dates, highlights, estimatedFare, warnings, assembledAt }`.
5. The workflow enters `AWAITING_APPROVAL`: it sends the itinerary to the user's confirmation endpoint and pauses.
6. If the user confirms, the workflow invokes the booking tool. A before-tool-call guardrail vets the booking payload against the stated budget and travel constraints before the tool fires. On guardrail failure the trip enters `BLOCKED`.
7. On successful booking the trip moves to `CONFIRMED`.
8. If the user declines, the trip moves to `DECLINED` with no booking action.
9. If either worker times out after 60 seconds, the workflow short-circuits: it assembles from whichever side returned, and the trip enters `DEGRADED`.

A `TripRequestSimulator` (TimedAction) drips a sample trip request every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TripCoordinator` | `AutonomousAgent` | Decomposes the trip request into worker tasks; assembles the final itinerary from worker outputs. | `TripWorkflow` | returns typed result to workflow |
| `DestinationScout` | `AutonomousAgent` | Researches destinations: attractions, travel advisories, visa requirements. Seeded "destination tool" returns canned results. | `TripWorkflow` | — |
| `FareAgent` | `AutonomousAgent` | Estimates flight and accommodation pricing for the requested dates and traveller count. Seeded "fare tool" returns canned results. | `TripWorkflow` | — |
| `TripWorkflow` | `Workflow` | Coordinates the parallel fan-out, itinerary assembly, HITL pause, before-tool-call guardrail, and booking step. | `TripEndpoint`, `TripRequestConsumer` | `TripEntity` |
| `TripEntity` | `EventSourcedEntity` | Holds the trip's lifecycle (planning → researching → awaiting-approval → confirmed / declined / degraded / blocked). | `TripWorkflow` | `TripView` |
| `TripRequestQueue` | `EventSourcedEntity` | Logs each submitted trip request for replay/audit. | `TripEndpoint`, `TripRequestSimulator` | `TripRequestConsumer` |
| `TripView` | `View` | List-of-trips read model. | `TripEntity` events | `TripEndpoint` |
| `TripRequestConsumer` | `Consumer` | Listens to `TripRequestQueue` events and starts a workflow per submission. | `TripRequestQueue` events | `TripWorkflow` |
| `TripRequestSimulator` | `TimedAction` | Drips a sample trip request every 90 s. | scheduler | `TripRequestQueue` |
| `TripEndpoint` | `HttpEndpoint` | `/api/trips/*` — submit, get, list, confirm/decline, SSE. | — | `TripView`, `TripRequestQueue`, `TripEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TripRequest(String destination, String departureDateIso, String returnDateIso,
                   int travellerCount, int budgetUsd, String requestedBy) {}

record DestinationBrief(String destination, List<String> highlights,
                        List<String> advisories, String visaRequirement,
                        Instant researchedAt) {}

record FareEstimate(int flightFareUsd, int accommodationFareUsd,
                    int totalFareUsd, String currency, Instant estimatedAt) {}

record TripPlan(String destinationQuery, String fareQuery) {}

record TripItinerary(String destination, String departureDateIso, String returnDateIso,
                     List<String> highlights, List<String> advisories,
                     String visaRequirement, FareEstimate fare,
                     Optional<String> warnings, Instant assembledAt) {}

record Trip(
    String tripId,
    String destination,
    String departureDateIso,
    String returnDateIso,
    int travellerCount,
    int budgetUsd,
    TripStatus status,
    Optional<DestinationBrief> destinationBrief,
    Optional<FareEstimate> fareEstimate,
    Optional<TripItinerary> itinerary,
    Optional<String> bookingReference,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TripStatus { PLANNING, RESEARCHING, AWAITING_APPROVAL, CONFIRMED, DECLINED, DEGRADED, BLOCKED }
```

### Events (on `TripEntity`)

`TripCreated`, `ResearchStarted`, `DestinationBriefAttached`, `FareEstimateAttached`,
`ItineraryAssembled`, `ApprovalRequested`, `TripConfirmed`, `TripDeclined`,
`TripDegraded`, `TripBlocked`.

### Events (on `TripRequestQueue`)

`TripRequested { tripId, destination, departureDateIso, returnDateIso, travellerCount, budgetUsd, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/trips` — body `TripRequest` → `{ tripId }`. Starts a workflow.
- `GET /api/trips` — list all trips. Optional `?status=PLANNING|RESEARCHING|AWAITING_APPROVAL|CONFIRMED|DECLINED|DEGRADED|BLOCKED`.
- `GET /api/trips/{id}` — one trip.
- `POST /api/trips/{id}/confirm` — user confirms the itinerary; resumes the paused workflow.
- `POST /api/trips/{id}/decline` — user declines; workflow ends with DECLINED.
- `GET /api/trips/sse` — server-sent events stream of every trip change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Travel Concierge"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a trip request (destination, dates, budget, traveller count), live list of trips with status pills, expand-row to see destination brief + fare estimate + assembled itinerary + confirm/decline buttons when status is AWAITING_APPROVAL.

Browser title: `<title>Akka Sample: Travel Concierge</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `TripCoordinator`): vets the booking action payload against the user's stated budget and travel constraints before any booking tool is invoked. Blocking. Failure → `BLOCKED`.
- **H1 — human-in-the-loop confirmation** (`application` HITL on `TripWorkflow`): the workflow pauses after itinerary assembly and waits for `POST /api/trips/{id}/confirm` or `.../decline` before invoking the booking tool. The pause is enforced in the workflow step, not the UI.

## 9. Agent prompts

- `TripCoordinator` → `prompts/trip-coordinator.md`. Decomposes the trip request into worker queries; later assembles the itinerary from worker results.
- `DestinationScout` → `prompts/destination-scout.md`. Researches destination facts; returns `DestinationBrief`.
- `FareAgent` → `prompts/fare-agent.md`. Estimates pricing; returns `FareEstimate`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a trip request; trip progresses PLANNING → RESEARCHING → AWAITING_APPROVAL within 60 s; confirm it → CONFIRMED; UI reflects each transition via SSE.
2. **J2** — When the trip is AWAITING_APPROVAL, decline it → DECLINED; no booking tool call is issued.
3. **J3** — Set budget below the fare estimate; confirm the trip → guardrail blocks the booking → BLOCKED.
4. **J4** — Inject a DestinationScout timeout (1 s); trip enters DEGRADED with the FareEstimate alone.
5. **J5** — Simulator drips a trip every 90 s; App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named travel-concierge demonstrating the
delegation-supervisor-workers × planning-travel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-planning-travel-travel-concierge.
Java package io.akka.samples.travelconcierge. Akka 3.6.0. HTTP port 9381.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TripCoordinator — definition() with
    capability(TaskAcceptance.of(DECOMPOSE_TRIP).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(ASSEMBLE_ITINERARY).maxIterationsPerTask(3)).
    System prompt loaded from prompts/trip-coordinator.md.
    Returns TripPlan{destinationQuery, fareQuery} for DECOMPOSE_TRIP and
    TripItinerary{destination, departureDateIso, returnDateIso, highlights,
    advisories, visaRequirement, fare, warnings, assembledAt} for ASSEMBLE_ITINERARY.
  * DestinationScout — capability(TaskAcceptance.of(SCOUT_DESTINATION).maxIterationsPerTask(3)).
    System prompt from prompts/destination-scout.md.
    Returns DestinationBrief{destination, highlights: List<String>, advisories: List<String>,
    visaRequirement, researchedAt}.
  * FareAgent — capability(TaskAcceptance.of(ESTIMATE_FARE).maxIterationsPerTask(2)).
    System prompt from prompts/fare-agent.md.
    Returns FareEstimate{flightFareUsd, accommodationFareUsd, totalFareUsd, currency, estimatedAt}.

- 1 Workflow TripWorkflow with steps:
  decomposeStep -> [parallel] scoutStep, fareStep -> joinStep -> assembleStep ->
  hitlStep -> guardrailStep -> bookStep -> emitStep.
  decomposeStep calls forAutonomousAgent(TripCoordinator.class, DECOMPOSE_TRIP).
  scoutStep and fareStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(TripWorkflow::scoutStep, ofSeconds(60)) and
  stepTimeout(TripWorkflow::fareStep, ofSeconds(60)). On either timeout, transition to
  degradeStep that assembles from whichever side returned, then ends with TripDegraded.
  assembleStep calls forAutonomousAgent(TripCoordinator.class, ASSEMBLE_ITINERARY) with the
  merged inputs; give assembleStep a 90s stepTimeout.
  hitlStep pauses the workflow; it emits ApprovalRequested on TripEntity and waits for a
  signal from TripEndpoint (POST /api/trips/{id}/confirm or .../decline). On decline, end
  with TripDeclined. On confirm, continue to guardrailStep.
  guardrailStep vets the booking payload against budget and travel constraints; on failure,
  end with TripBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TripEntity holding state Trip{tripId, destination,
  departureDateIso, returnDateIso, travellerCount, budgetUsd, TripStatus,
  Optional<DestinationBrief> destinationBrief, Optional<FareEstimate> fareEstimate,
  Optional<TripItinerary> itinerary, Optional<String> bookingReference,
  Optional<String> failureReason, Instant createdAt, Optional<Instant> finishedAt}.
  TripStatus enum: PLANNING, RESEARCHING, AWAITING_APPROVAL, CONFIRMED, DECLINED,
  DEGRADED, BLOCKED.
  Events: TripCreated, ResearchStarted, DestinationBriefAttached, FareEstimateAttached,
  ItineraryAssembled, ApprovalRequested, TripConfirmed, TripDeclined, TripDegraded,
  TripBlocked.
  Commands: createTrip, startResearch, attachDestinationBrief, attachFareEstimate,
  assembleItinerary, requestApproval, confirmTrip, declineTrip, degradeTrip, blockTrip,
  getTrip.
  emptyState() returns Trip.initial("", "") with no commandContext() reference.

- 1 EventSourcedEntity TripRequestQueue with command enqueueTripRequest(tripId, destination,
  departureDateIso, returnDateIso, travellerCount, budgetUsd, requestedBy) emitting
  TripRequested{tripId, destination, departureDateIso, returnDateIso, travellerCount,
  budgetUsd, requestedBy, submittedAt}.

- 1 View TripView with row type TripRow (mirrors Trip minus heavy nested payloads; every
  nullable field is Optional<T>). Table updater consumes TripEntity events. ONE query
  getAllTrips SELECT * AS trips FROM trip_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TripRequestConsumer subscribed to TripRequestQueue events; on TripRequested
  starts a TripWorkflow with the tripId as the workflow id.

- 2 TimedActions:
  * TripRequestSimulator — every 90 s, reads the next line from
    src/main/resources/sample-events/trip-requests.jsonl and calls
    TripRequestQueue.enqueueTripRequest.
  * No EvalSampler in this baseline — governance is guardrail + HITL only.

- 2 HttpEndpoints:
  * TripEndpoint at /api with POST /trips, GET /trips, GET /trips/{id},
    POST /trips/{id}/confirm, POST /trips/{id}/decline,
    GET /trips/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TripTasks.java declaring four Task<R> constants: DECOMPOSE_TRIP (TripPlan),
  SCOUT_DESTINATION (DestinationBrief), ESTIMATE_FARE (FareEstimate),
  ASSEMBLE_ITINERARY (TripItinerary).
- Domain records TripPlan, DestinationBrief, FareEstimate, TripItinerary, TripRequest.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9381 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/trip-requests.jsonl with 8 canned trip request lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 before-tool-call guardrail,
  H1 HITL application) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = travel and hospitality,
  decisions.authority_level = recommend-only, data.data_classes.pii = true (names +
  travel dates), capabilities.user_facing_interaction = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/trip-coordinator.md, prompts/destination-scout.md, prompts/fare-agent.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Travel Concierge", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no
  ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills + confirm/decline buttons when AWAITING_APPROVAL).
  Browser title exactly: <title>Akka Sample: Travel Concierge</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via the
  conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://; recorded in
        .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the
  ModelProvider interface with a per-agent dispatch on the agent class name (or Task<R> id).
  Each agent's branch reads a JSON file from src/main/resources/mock-responses/<agent>.json
  (trip-coordinator.json, destination-scout.json, fare-agent.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    trip-coordinator.json — list of either TripPlan or TripItinerary objects.
      4–6 TripPlan entries (destinationQuery + fareQuery pairs) and 4–6 TripItinerary
      entries (each with a realistic destination, 3–5 highlights, 1–2 advisories, a
      visaRequirement string, a fare object, warnings = null, assembledAt).
    destination-scout.json — 4–6 DestinationBrief entries, each with 3–5 highlights,
      0–2 advisories, and a plain-English visaRequirement sentence.
    fare-agent.json — 4–6 FareEstimate entries with plausible USD amounts and
      currency = "USD".
- A MockModelProvider.seedFor(tripId) helper makes selection deterministic per trip id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s assemble); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion TripTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9381 in application.conf.
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
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
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
