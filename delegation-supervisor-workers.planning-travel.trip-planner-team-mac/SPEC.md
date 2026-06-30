# SPEC — trip-planner-team-mac

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Trip Planner Multi-Agent.
**One-line pitch:** Describe a trip; a supervisor delegates destination research, itinerary drafting, and booking preparation to three specialists in parallel, then waits for your approval before confirming anything.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their results, gates them through a **before-tool-call guardrail** that fires before any booking payload is executed, and then pauses at a **human-in-the-loop** checkpoint so the user approves or rejects the proposed plan. The blueprint also demonstrates an **eval-event** that samples the supervisor's compilation decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a trip request (destination, dates, traveller count, budget).

1. The system creates a `Trip` record in `PLANNING` and starts a `TripWorkflow`.
2. The TripSupervisor decomposes the request into three parallel work items: a destination research query, an itinerary brief, and a booking preparation task.
3. The workflow forks: all three agents run concurrently. Each returns a typed payload.
4. The TripSupervisor compiles the three outputs into a `TripPlan { itinerary, destinationNotes, bookingProposals, compilationNotes }`.
5. A **before-tool-call guardrail** checks that no booking proposal exceeds the stated budget and that all required fields are present. If it fails, the trip enters `BLOCKED`.
6. If guardrail passes, the trip enters `AWAITING_APPROVAL` and a unique approval token is written to `ApprovalEntity`. The App UI shows the draft plan with Approve / Reject buttons.
7. The user clicks Approve → `POST /api/trips/{id}/approve` → the workflow resumes and the trip enters `CONFIRMED`. The user clicks Reject → trip enters `REJECTED`. Either way no reservation is made in this sample (the booking tools are modelled in-process).
8. If any worker times out after 60 seconds, the workflow short-circuits: TripSupervisor compiles from whichever sides returned, and the trip enters `DEGRADED`.

A `TripRequestSimulator` (TimedAction) drips a sample request every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TripSupervisor` | `AutonomousAgent` | Decomposes the trip request, compiles the final plan from the three worker outputs. | `TripWorkflow` | returns typed result to workflow |
| `DestinationResearcher` | `AutonomousAgent` | Gathers destination facts (climate, visa, highlights, local tips). | `TripWorkflow` | — |
| `ItineraryPlanner` | `AutonomousAgent` | Drafts a day-by-day itinerary from destination notes. | `TripWorkflow` | — |
| `BookingAgent` | `AutonomousAgent` | Constructs flight and accommodation reservation payloads; never executes without guardrail + HITL clearance. | `TripWorkflow` | — |
| `TripWorkflow` | `Workflow` | Coordinates the parallel fan-out, the booking guardrail, the HITL pause, and the confirm/reject/degrade branches. | `TripEndpoint`, `TripRequestConsumer` | `TripEntity`, `ApprovalEntity` |
| `TripEntity` | `EventSourcedEntity` | Holds the trip lifecycle (planning → in-progress → awaiting-approval → confirmed / rejected / degraded / blocked). | `TripWorkflow` | `TripView` |
| `ApprovalEntity` | `EventSourcedEntity` | Holds the approval token, decision, and audit trail for HITL. | `TripWorkflow`, `TripEndpoint` | — |
| `TripRequestQueue` | `EventSourcedEntity` | Logs each submitted request for replay/audit. | `TripEndpoint`, `TripRequestSimulator` | `TripRequestConsumer` |
| `TripView` | `View` | List-of-trips read model. | `TripEntity` events | `TripEndpoint` |
| `TripRequestConsumer` | `Consumer` | Listens to `TripRequestQueue` events and starts a workflow per submission. | `TripRequestQueue` events | `TripWorkflow` |
| `TripRequestSimulator` | `TimedAction` | Drips a sample trip request every 90 s. | scheduler | `TripRequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one confirmed plan every 10 minutes for quality scoring; emits a `PlanEvalScored` event. | scheduler | `TripEntity` |
| `TripEndpoint` | `HttpEndpoint` | `/api/trips/*` — submit, get, list, SSE, approve, reject. | — | `TripView`, `TripRequestQueue`, `TripEntity`, `ApprovalEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TripRequest(
    String destination,
    String startDate,
    String endDate,
    int travellerCount,
    double budgetUsd,
    String requestedBy
) {}

record DestinationNotes(
    String climate,
    String visaRequirements,
    List<String> highlights,
    List<String> localTips,
    Instant researchedAt
) {}

record ItineraryDay(int day, String title, String description, List<String> activities) {}
record Itinerary(List<ItineraryDay> days, String overallTheme, Instant draftedAt) {}

record BookingProposal(
    String type,
    String description,
    double estimatedCostUsd,
    String referenceCode,
    Instant preparedAt
) {}

record TripPlan(
    Itinerary itinerary,
    DestinationNotes destinationNotes,
    List<BookingProposal> bookingProposals,
    String compilationNotes,
    Instant compiledAt
) {}

record Trip(
    String tripId,
    TripRequest request,
    TripStatus status,
    Optional<DestinationNotes> destinationNotes,
    Optional<Itinerary> itinerary,
    Optional<List<BookingProposal>> bookingProposals,
    Optional<TripPlan> plan,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TripStatus {
    PLANNING, IN_PROGRESS, AWAITING_APPROVAL, CONFIRMED, REJECTED, DEGRADED, BLOCKED
}
```

### Events (on `TripEntity`)

`TripCreated`, `DestinationResearched`, `ItineraryDrafted`, `BookingsPrepared`, `PlanCompiled`, `ApprovalRequested`, `TripConfirmed`, `TripRejected`, `TripDegraded`, `TripBlocked`, `PlanEvalScored`.

### Events (on `ApprovalEntity`)

`ApprovalTokenIssued { tripId, token, issuedAt }`, `ApprovalDecisionRecorded { tripId, decision, decidedAt }`.

### Events (on `TripRequestQueue`)

`TripRequestSubmitted { tripId, destination, startDate, endDate, travellerCount, budgetUsd, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/trips` — body `{ destination, startDate, endDate, travellerCount, budgetUsd }` → `{ tripId }`. Starts a workflow.
- `GET /api/trips` — list all trips. Optional `?status=PLANNING|IN_PROGRESS|AWAITING_APPROVAL|CONFIRMED|REJECTED|DEGRADED|BLOCKED`.
- `GET /api/trips/{id}` — one trip.
- `GET /api/trips/sse` — server-sent events stream of every trip change.
- `POST /api/trips/{id}/approve` — body `{ token }` → resumes the workflow with approval.
- `POST /api/trips/{id}/reject` — body `{ token, reason }` → resumes the workflow with rejection.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Trip Planner Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a trip request (destination, dates, traveller count, budget), live list of trips with status pills, expand-row to see destination notes + itinerary + booking proposals + plan summary + eval score; Approve / Reject buttons on `AWAITING_APPROVAL` rows.

Browser title: `<title>Akka Sample: Trip Planner Multi-Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `TripWorkflow.guardrailStep`): checks that no booking proposal exceeds the stated budget and that all required booking fields are present. Blocking. Failure → `BLOCKED`.
- **H1 — HITL approval checkpoint** (`application`-layer): after the guardrail passes, the workflow pauses and issues an `ApprovalEntity` token. The plan is never confirmed without an explicit `POST /api/trips/{id}/approve`. Rejection routes to `REJECTED` with no booking made.

## 9. Agent prompts

- `TripSupervisor` → `prompts/trip-supervisor.md`. Decomposes a trip request into parallel work items; later compiles the three specialist outputs into a `TripPlan`.
- `DestinationResearcher` → `prompts/destination-researcher.md`. Gathers destination facts; returns `DestinationNotes`.
- `ItineraryPlanner` → `prompts/itinerary-planner.md`. Drafts a day-by-day itinerary; returns `Itinerary`.
- `BookingAgent` → `prompts/booking-agent.md`. Constructs reservation payloads; returns `List<BookingProposal>`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a trip request; plan progresses `PLANNING → IN_PROGRESS → AWAITING_APPROVAL` within 90 s; UI reflects each transition via SSE; user approves → `CONFIRMED`.
2. **J2** — Inject a worker timeout (set `ItineraryPlanner` timeout to 1 s); trip enters `DEGRADED` with whichever partial output came back.
3. **J3** — Submit a request whose budget is too low for the generated proposals; guardrail rejects; trip enters `BLOCKED`.
4. **J4** — User rejects the draft; trip enters `REJECTED`; no booking is made.
5. **J5** — Wait for a confirmed plan; `EvalSampler` records a score within 10 minutes; App UI row shows the score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named trip-planner-team-mac demonstrating the
delegation-supervisor-workers × planning-travel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-planning-travel-trip-planner-team-mac.
Java package io.akka.samples.tripplannermultiagent. Akka 3.6.0. HTTP port 9621.

Components to wire (exactly):
- 4 AutonomousAgents:
  * TripSupervisor — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(COMPILE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/trip-supervisor.md. Returns TripDecomposition{researchQuery, itineraryBrief,
    bookingScope} for DECOMPOSE and TripPlan{itinerary, destinationNotes, bookingProposals,
    compilationNotes, compiledAt} for COMPILE.
  * DestinationResearcher — capability(TaskAcceptance.of(RESEARCH_DESTINATION).maxIterationsPerTask(3)).
    System prompt from prompts/destination-researcher.md. Returns DestinationNotes{climate,
    visaRequirements, highlights: List<String>, localTips: List<String>, researchedAt}.
  * ItineraryPlanner — capability(TaskAcceptance.of(PLAN_ITINERARY).maxIterationsPerTask(3)).
    System prompt from prompts/itinerary-planner.md. Returns Itinerary{days: List<ItineraryDay
    {day, title, description, activities: List<String>}>, overallTheme, draftedAt}.
  * BookingAgent — capability(TaskAcceptance.of(PREPARE_BOOKINGS).maxIterationsPerTask(2)).
    System prompt from prompts/booking-agent.md. Returns List<BookingProposal{type, description,
    estimatedCostUsd, referenceCode, preparedAt}>.

- 1 Workflow TripWorkflow with steps:
  decomposeStep -> [parallel] researchStep, planStep, bookStep -> joinStep -> compileStep
  -> guardrailStep -> approvalStep (pause) -> confirmStep or rejectStep.
  decomposeStep calls forAutonomousAgent(TripSupervisor.class, DECOMPOSE).
  researchStep, planStep, and bookStep run in parallel (CompletionStage allOf); each governed by
  WorkflowSettings.builder().stepTimeout(TripWorkflow::researchStep, ofSeconds(60)),
  stepTimeout(TripWorkflow::planStep, ofSeconds(60)), stepTimeout(TripWorkflow::bookStep,
  ofSeconds(60)). On any timeout, transition to a degradeStep that calls compileStep with
  whichever sides returned, then ends with TripDegraded.
  compileStep calls forAutonomousAgent(TripSupervisor.class, COMPILE) with the merged inputs;
  give compileStep a 90s stepTimeout. guardrailStep evaluates each booking proposal against
  the stated budgetUsd: if any single proposal exceeds 80% of budget or the total exceeds
  budget, the trip enters BLOCKED. approvalStep persists an ApprovalEntity token and pauses
  the workflow (no timeout); it resumes when the endpoint delivers an approve or reject signal.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TripEntity holding state Trip{tripId, TripRequest request, TripStatus,
  Optional<DestinationNotes>, Optional<Itinerary>, Optional<List<BookingProposal>>,
  Optional<TripPlan>, Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  TripStatus enum: PLANNING, IN_PROGRESS, AWAITING_APPROVAL, CONFIRMED, REJECTED, DEGRADED,
  BLOCKED. Events: TripCreated, DestinationResearched, ItineraryDrafted, BookingsPrepared,
  PlanCompiled, ApprovalRequested, TripConfirmed, TripRejected, TripDegraded, TripBlocked,
  PlanEvalScored. Commands: createTrip, attachDestination, attachItinerary, attachBookings,
  compilePlan, requestApproval, confirmTrip, rejectTrip, degrade, block, recordEval, getTrip.
  emptyState() returns Trip.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity ApprovalEntity with command issueToken(tripId) emitting
  ApprovalTokenIssued{tripId, token, issuedAt} and command recordDecision(tripId, decision,
  reason) emitting ApprovalDecisionRecorded{tripId, decision, decidedAt}.

- 1 EventSourcedEntity TripRequestQueue with command enqueueRequest(TripRequest) emitting
  TripRequestSubmitted{tripId, destination, startDate, endDate, travellerCount, budgetUsd,
  requestedBy, submittedAt}.

- 1 View TripView with row type TripRow (mirrors Trip minus heavy nested payloads; every
  nullable field is Optional<T>). Table updater consumes TripEntity events. ONE query
  getAllTrips SELECT * AS trips FROM trip_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer TripRequestConsumer subscribed to TripRequestQueue events; on
  TripRequestSubmitted starts a TripWorkflow with the tripId as the workflow id.

- 2 TimedActions:
  * TripRequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/trip-requests.jsonl and calls
    TripRequestQueue.enqueueRequest.
  * EvalSampler — every 10 minutes, queries TripView.getAllTrips, picks the oldest
    CONFIRMED trip without an evalScore, runs a 1–5 rubric judge over the compiled
    plan, then calls TripEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * TripEndpoint at /api with POST /trips, GET /trips, GET /trips/{id},
    GET /trips/sse, POST /trips/{id}/approve, POST /trips/{id}/reject, and the
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TripTasks.java declaring five Task<R> constants: DECOMPOSE (TripDecomposition),
  RESEARCH_DESTINATION (DestinationNotes), PLAN_ITINERARY (Itinerary),
  PREPARE_BOOKINGS (List<BookingProposal>), COMPILE (TripPlan).
- Domain records TripRequest, DestinationNotes, ItineraryDay, Itinerary, BookingProposal,
  TripPlan, TripDecomposition.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9621 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/trip-requests.jsonl with 8 canned request lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 before-tool-call guardrail,
  H1 HITL application) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = travel planning, decisions.
  authority_level = recommend-only, data.pii = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/trip-supervisor.md, prompts/destination-researcher.md,
  prompts/itinerary-planner.md, prompts/booking-agent.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: Trip Planner Multi-Agent", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no
  ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills; Approve/Reject buttons on AWAITING_APPROVAL rows).
  Browser title exactly: <title>Akka Sample: Trip Planner Multi-Agent</title>. No subtitle
  on the Overview tab.

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
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    trip-supervisor.json — list of TripDecomposition entries (4–6) and
      TripPlan entries (4–6). Each TripPlan has a 60–120 word compilationNotes,
      a 3–5 day itinerary, 2–3 destination notes highlights, and 2–3
      booking proposals.
    destination-researcher.json — 4–6 DestinationNotes entries, each with
      climate (2 sentences), visaRequirements (1 sentence), 3–5 highlights,
      2–4 localTips.
    itinerary-planner.json — 4–6 Itinerary entries, each with 3–5
      ItineraryDay records and a one-sentence overallTheme.
    booking-agent.json — 4–6 List<BookingProposal> entries, each with 2–3
      proposals (one flight, one accommodation); estimatedCostUsd values
      well within a typical budgetUsd.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s compile); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion TripTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9621 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index.
- Parallel workflow steps use CompletionStage allOf, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, simplest, complex, Akka SDK in narrative,
  T1/T2/T3/T4, deferred, marketing tone.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
