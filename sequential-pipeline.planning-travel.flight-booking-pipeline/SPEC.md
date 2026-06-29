# SPEC — flight-booking-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Flight Booking.
**One-line pitch:** A user submits an origin, destination, and travel date; one `BookingAgent` walks the request through three task phases — **SEARCH** for available flights, **SELECT** the best seat on the chosen flight, **BOOK** the reservation — with a mandatory user-confirmation gate before the irreversible booking write executes.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a travel-booking domain. One `BookingAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the SEARCH task's typed output (`FlightOfferSet`) becomes the SELECT task's instruction context; the SELECT task's typed output (`SeatSelection`) becomes the BOOK task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`SEARCH` / `SELECT` / `BOOK`) and the current `BookingEntity` status. A BOOK-phase tool called while the entity has not yet recorded `BookingConfirmed` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. The guardrail enforces two distinct preconditions: phase order (the dependency contract) and user confirmation (the irreversible-spend gate). Both are checked at the same `before-tool-call` hook so neither can be bypassed independently.
- A **human-in-the-loop (HITL) application gate** runs between `selectStep` and `bookStep`. After `SeatSelected` is recorded, the workflow pauses at `confirmStep` and waits for a `POST /api/bookings/{id}/confirm` call from the user. Only after `BookingConfirmed` lands does the workflow proceed to `bookStep`. The `BookingGuardrail` enforces this: `commitReservation` requires `status ∈ {CONFIRMED}` AND `seatSelection.isPresent()`. No confirmation, no booking.

The blueprint shows that the sequential-pipeline pattern is not just about chaining LLM calls — the task-boundary handoffs are the right cut for both dependency enforcement and the irreversible-spend protection gate.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in an **origin**, **destination**, and **travel date** (or picks one of three seeded routes — `SFO → JFK, 2026-09-15`, `LHR → CDG, 2026-09-20`, `NRT → SIN, 2026-09-25`).
2. The user clicks **Find flights**. The UI POSTs to `/api/bookings` and receives a `bookingId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `SEARCHING` — the workflow has started `searchStep` and the agent has been handed the SEARCH task.
4. Within ~10–20 s the card reaches `SELECTING` — the typed `FlightOfferSet` is visible in the card detail (a table of flight offers with carrier, flight number, departure/arrival times, and fare). The agent's SEARCH task returned; the workflow recorded `FlightsFound` and ran the SELECT task.
5. Within ~10–20 s more the card reaches `AWAITING_CONFIRMATION` — the `SeatSelection` is visible (flight number, seat code, cabin class, price). A yellow **Confirm booking** button appears.
6. The user reviews the itinerary and clicks **Confirm booking**. The UI POSTs to `/api/bookings/{id}/confirm`.
7. The card transitions to `BOOKING`. Within ~10–20 s it reaches `COMMITTED`. The right pane now shows the full `BookingConfirmation` — confirmation code, passenger name placeholder, itinerary details — plus a green "Booking committed" badge.
8. The user can submit another route; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BookingEndpoint` | `HttpEndpoint` | `/api/bookings/*` — submit, confirm, list, get, SSE; serves `/api/metadata/*`. | — | `BookingEntity`, `BookingView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BookingEntity` | `EventSourcedEntity` | Per-booking lifecycle: created → searching → search_done → selecting → seat_selected → awaiting_confirmation → confirmed → booking → committed. Source of truth. | `BookingEndpoint`, `BookingPipelineWorkflow` | `BookingView` |
| `BookingPipelineWorkflow` | `Workflow` | One workflow per booking. Steps: `searchStep` → `selectStep` → `confirmStep` → `bookStep`. Each step calls one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `confirmStep` waits on `BookingConfirmed`. | started by `BookingEndpoint` after `CREATED` | `BookingAgent`, `BookingEntity` |
| `BookingAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `BookingTasks.java`: `SEARCH_FLIGHTS` → `FlightOfferSet`, `SELECT_SEAT` → `SeatSelection`, `COMMIT_BOOKING` → `BookingConfirmation`. Each task is registered with the phase-appropriate function tools. | invoked by `BookingPipelineWorkflow` | returns typed results |
| `SearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchFlights(origin, destination, date)` and `getFareDetails(flightId)`. Reads from `src/main/resources/sample-data/routes/*.json` for deterministic offline output. | called from SEARCH task | returns `List<FlightOffer>` |
| `SelectTools` | function-tools class | Implements `getSeatMap(flightId)` and `rankSeats(seatMap, preferences)`. Pure in-memory transformations. | called from SELECT task | returns `SeatMap` / `RankedSeat` |
| `BookTools` | function-tools class | Implements `commitReservation(flightId, seatCode, passengerRef)` and `getConfirmationDetails(reservationId)`. | called from BOOK task | returns `ReservationId` / `BookingConfirmation` |
| `BookingGuardrail` | `before-tool-call` guardrail (registered on `BookingAgent`) | Reads the in-flight task's declared phase and the current `BookingEntity` status. Rejects any tool call whose phase precondition has not been satisfied, and additionally rejects `commitReservation` until `BookingConfirmed` is on record. | every tool call on every task | accept / structured-reject |
| `BookingView` | `View` | Read model: one row per booking for the UI. | `BookingEntity` events | `BookingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record FlightOffer(
    String flightId,
    String carrier,
    String flightNumber,
    String origin,
    String destination,
    Instant departureAt,
    Instant arrivalAt,
    String cabinClass,
    int fareUsdCents
) {}

record FlightOfferSet(List<FlightOffer> offers, Instant searchedAt) {}

record SeatMap(String flightId, List<SeatInfo> seats) {}

record SeatInfo(String seatCode, String cabinClass, boolean available, int upgradeCostUsdCents) {}

record RankedSeat(String seatCode, String cabinClass, int totalCostUsdCents, String rationale) {}

record SeatSelection(
    String flightId,
    String carrier,
    String flightNumber,
    String origin,
    String destination,
    Instant departureAt,
    Instant arrivalAt,
    String seatCode,
    String cabinClass,
    int totalCostUsdCents,
    Instant selectedAt
) {}

record ReservationId(String reservationId) {}

record BookingConfirmation(
    String confirmationCode,
    String passengerRef,
    String flightId,
    String carrier,
    String flightNumber,
    String origin,
    String destination,
    Instant departureAt,
    String seatCode,
    String cabinClass,
    int totalCostUsdCents,
    Instant committedAt
) {}

record BookingRecord(
    String bookingId,
    Optional<String> origin,
    Optional<String> destination,
    Optional<String> travelDate,
    Optional<FlightOfferSet> flightOffers,
    Optional<SeatSelection> seatSelection,
    Optional<BookingConfirmation> confirmation,
    BookingStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum BookingStatus {
    CREATED, SEARCHING, SEARCH_DONE, SELECTING, SEAT_SELECTED,
    AWAITING_CONFIRMATION, CONFIRMED, BOOKING, COMMITTED, FAILED
}
```

Events on `BookingEntity`: `BookingCreated`, `SearchStarted`, `FlightsFound`, `SelectStarted`, `SeatSelected`, `AwaitingConfirmation`, `BookingConfirmed`, `BookStarted`, `BookingCommitted`, `GuardrailRejected`, `BookingFailed`.

Every nullable lifecycle field on the `BookingRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/bookings` — body `{ origin, destination, travelDate }` → `{ bookingId }`.
- `POST /api/bookings/{id}/confirm` — no body → `{ bookingId, status: "CONFIRMED" }`.
- `GET /api/bookings` — list all bookings, newest-first.
- `GET /api/bookings/{id}` — one booking.
- `GET /api/bookings/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Flight Booking</title>`.

The App UI tab is a two-column layout: a left rail with the live list of bookings (status pill + route + age) and a right pane with the selected booking's detail — route, flight offers table, seat selection summary, itinerary confirmation view, and a guardrail-rejection log strip if any phase-gate rejections occurred. The **Confirm booking** button appears only when the booking is in `AWAITING_CONFIRMATION` status.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — HITL application gate (user confirmation)**: The workflow pauses at `confirmStep` after `SeatSelected` is recorded. The UI renders a **Confirm booking** button; no booking proceeds without it. `BookingGuardrail` enforces this at the tool level: `commitReservation` requires `status ∈ {CONFIRMED}` AND `seatSelection.isPresent()`. If the agent loop somehow calls `commitReservation` before the user has confirmed, the guardrail rejects the call and records a `GuardrailRejected{phase: "BOOK", tool: "commitReservation", reason: "hitl-violation: booking not confirmed by user"}` event. The workflow waits; the user's `/confirm` call is the only path forward. This gate is the core reason the sequential pipeline stops at `AWAITING_CONFIRMATION` and does not self-approve.
- **G1 — `before-tool-call` guardrail (phase-gate + spend protection)**: `BookingGuardrail` is registered on `BookingAgent` and runs before every tool call. Phase-order rules: `SEARCH` tools require `status ∈ {CREATED, SEARCHING}`; `SELECT` tools require `status ∈ {SEARCH_DONE, SELECTING}` AND `flightOffers.isPresent()`; `BOOK` tools require `status ∈ {CONFIRMED, BOOKING}` AND `seatSelection.isPresent()` AND `flightOffers.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and records a `GuardrailRejected` event. The agent loop retries within its 4-iteration budget.

## 9. Agent prompts

- `BookingAgent` → `prompts/booking-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded route `SFO → JFK, 2026-09-15`; within 60 s the booking reaches `AWAITING_CONFIRMATION` with ≥ 2 flight offers and a selected seat; user confirms; booking reaches `COMMITTED` with a non-empty confirmation code.
2. **J2** — The agent's first iteration on a booking calls `commitReservation` before `BookingConfirmed` has been recorded (mock LLM path). `BookingGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the workflow remains in `AWAITING_CONFIRMATION`; the UI shows the confirmation button; user confirms; booking eventually reaches `COMMITTED`.
3. **J3** — The agent calls a SELECT-phase tool (`getSeatMap`) during the SEARCH phase (forced via the mock LLM). `BookingGuardrail` rejects the call with a `phase-violation` error; the rejection event appears in the rejection-log strip; the agent retries with a SEARCH-phase tool; the pipeline completes.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-booking trace (logged at `INFO`); the SEARCH task's log shows only SEARCH-tool calls, the SELECT task's log shows only SELECT-tool calls, and the BOOK task's log shows only BOOK-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named flight-booking-pipeline demonstrating the sequential-pipeline x
planning-travel cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-planning-travel-flight-booking-pipeline. Java package
io.akka.samples.flightbooking. Akka 3.6.0. HTTP port 9149.

Components to wire (exactly):

- 1 AutonomousAgent BookingAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/booking-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  SEARCH, SELECT, and BOOK tool sets are ALL registered on the agent; phase gating and
  confirmation gating are the job of BookingGuardrail, NOT of conditional .tools(...) wiring.
  The before-tool-call guardrail (BookingGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow BookingPipelineWorkflow per bookingId with four steps:
  * searchStep — emits SearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(BookingAgent.class, "agent-" + bookingId).runSingleTask(
      TaskDef.instructions("Route: " + origin + " -> " + destination + "\nDate: " + travelDate
      + "\nPhase: SEARCH\nUse searchFlights and getFareDetails to find 3-5 flight offers.")
        .metadata("bookingId", bookingId)
        .metadata("phase", "SEARCH")
        .taskType(BookingTasks.SEARCH_FLIGHTS)
    ). Reads forTask(taskId).result(SEARCH_FLIGHTS) to get FlightOfferSet. Writes
    BookingEntity.recordFlights(flightOfferSet). WorkflowSettings.stepTimeout 60s.
  * selectStep — emits SelectStarted, then runSingleTask with TaskDef.instructions
    (formatSelectContext(flightOfferSet, origin, destination, travelDate)) and
    metadata.phase = "SELECT", taskType SELECT_SEAT. Writes
    BookingEntity.recordSeatSelection(seatSelection). stepTimeout 60s.
  * confirmStep — emits AwaitingConfirmation on the entity, then waits using
    Workflow.asyncResult: suspends until BookingConfirmed lands on the entity (polled via
    componentClient.forEventSourcedEntity). stepTimeout 3600s (1 hour — human must confirm).
  * bookStep — emits BookStarted, then runSingleTask with TaskDef.instructions
    (formatBookContext(seatSelection)) and metadata.phase = "BOOK", taskType COMMIT_BOOKING.
    Writes BookingEntity.recordConfirmation(bookingConfirmation). stepTimeout 60s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(BookingPipelineWorkflow::error). The error step writes
  BookingFailed and ends.

- 1 EventSourcedEntity BookingEntity (one per bookingId). State BookingRecord{bookingId,
  origin: Optional<String>, destination: Optional<String>, travelDate: Optional<String>,
  flightOffers: Optional<FlightOfferSet>, seatSelection: Optional<SeatSelection>,
  confirmation: Optional<BookingConfirmation>, status: BookingStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  BookingStatus enum: CREATED, SEARCHING, SEARCH_DONE, SELECTING, SEAT_SELECTED,
  AWAITING_CONFIRMATION, CONFIRMED, BOOKING, COMMITTED, FAILED. Events:
  BookingCreated{origin, destination, travelDate}, SearchStarted, FlightsFound{flightOffers},
  SelectStarted, SeatSelected{seatSelection}, AwaitingConfirmation, BookingConfirmed,
  BookStarted, BookingCommitted{confirmation}, GuardrailRejected{phase, tool, reason,
  rejectedAt}, BookingFailed{reason}. Commands: create, startSearch, recordFlights,
  startSelect, recordSeatSelection, markAwaitingConfirmation, confirm, startBook,
  recordConfirmation, recordGuardrailRejection, fail, getBooking. emptyState() returns
  BookingRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View BookingView with row type BookingRow that mirrors BookingRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes BookingEntity events. ONE
  query getAllBookings: SELECT * AS bookings FROM booking_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * BookingEndpoint at /api with:
    POST /bookings (body {origin, destination, travelDate}; mints bookingId; calls
      BookingEntity.create(origin, destination, travelDate); then starts
      BookingPipelineWorkflow with id "pipeline-" + bookingId; returns {bookingId}),
    POST /bookings/{id}/confirm (calls BookingEntity.confirm(id); returns
      {bookingId, status: "CONFIRMED"}),
    GET /bookings (list from getAllBookings, sorted newest-first),
    GET /bookings/{id} (one row),
    GET /bookings/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- BookingTasks.java declaring three Task<R> constants:
    SEARCH_FLIGHTS = Task.name("Search flights").description("Find available flights on the
      given route and date by calling searchFlights and getFareDetails").
      resultConformsTo(FlightOfferSet.class);
    SELECT_SEAT = Task.name("Select seat").description("Pick the best available seat on the
      chosen flight by calling getSeatMap and rankSeats").resultConformsTo(SeatSelection.class);
    COMMIT_BOOKING = Task.name("Commit booking").description("Commit the selected seat by
      calling commitReservation and getConfirmationDetails").
      resultConformsTo(BookingConfirmation.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {SEARCH, SELECT, BOOK}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "searchFlights", phase = Phase.SEARCH)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- SearchTools.java — @FunctionTool searchFlights(String origin, String destination,
  String date) -> List<FlightOffer> reading from
  src/main/resources/sample-data/routes/*.json keyed by route+date; @FunctionTool
  getFareDetails(String flightId) -> FlightOffer reading the matching entry's full fare
  record.

- SelectTools.java — @FunctionTool getSeatMap(String flightId) -> SeatMap reading from
  the matching route file's seat inventory; @FunctionTool rankSeats(SeatMap seatMap,
  String preferences) -> RankedSeat (deterministic ranking — lowest total cost wins when
  preferences = "economy"; window seat in economy wins when preferences = "window"; 2-3
  ranked candidates typical).

- BookTools.java — @FunctionTool commitReservation(String flightId, String seatCode,
  String passengerRef) -> ReservationId (deterministic id = "res-" +
  sha1(flightId+seatCode+passengerRef).substring(0,8)); @FunctionTool
  getConfirmationDetails(String reservationId) -> BookingConfirmation (reads from the
  matching in-memory reservation registry seeded at startup).

- BookingGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the BookingEntity status by bookingId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("phase-violation: <tool> requires <precondition>,
  saw <status>") or Guardrail.reject("hitl-violation: booking not confirmed by user").
  On reject ALSO calls BookingEntity.recordGuardrailRejection(phase, tool, reason) so the
  rejection is visible in the UI's rejection-log strip and in the audit log.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9149 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/routes/*.json — three files keyed by seeded route+date,
  each carrying 3-5 FlightOffer entries and a SeatMap per flight with 10-20 seats, with
  deterministic content so SearchTools.searchFlights returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, G1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — sample domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (route + date are
  the only user inputs), decisions.authority_level = action-on-behalf-of-user (the booking
  commits a financial transaction), oversight.human_in_loop = true (user confirms before spend),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "premature-booking-write", "phase-violation",
  "duplicate-reservation", "seat-unavailable"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/booking-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Flight Booking", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of booking cards; right = selected-booking detail with route header, flight offers
  table, seat selection summary, booking confirmation view, confirm button, rejection-log
  strip). Browser title exactly: <title>Akka Sample: Flight Booking</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(bookingId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    search-flights.json — 5 FlightOfferSet entries, each with 3-5 FlightOffer items per
      seeded route. Each entry's tool_calls array contains 1-2 calls: 1 searchFlights(route,
      date) + 0-1 getFareDetails(flightId) calls. Plus 1 deliberately PHASE-VIOLATING entry
      whose tool_calls array starts with a getSeatMap(...) call (a SELECT-phase tool called
      during the SEARCH phase) — the guardrail rejects it, the mock then falls through to a
      normal search sequence. The mock should select the violating entry on the FIRST iteration
      of every 3rd booking (modulo seed) so J3 is reproducible.
    select-seat.json — 5 SeatSelection entries paired one-to-one with the search entries,
      each with a specific seatCode and cabinClass, with tool_calls containing getSeatMap +
      rankSeats in order.
    commit-booking.json — 5 BookingConfirmation entries paired one-to-one. Each carries a
      confirmationCode and full itinerary details, with tool_calls containing
      commitReservation + getConfirmationDetails. Plus 1 PREMATURE-COMMIT entry whose
      tool_calls array starts with commitReservation called before BookingConfirmed is on
      record — the guardrail rejects it with "hitl-violation"; J2 verifies this.
- A MockModelProvider.seedFor(bookingId) helper makes per-booking selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BookingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BookingTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (searchStep
  60s, selectStep 60s, confirmStep 3600s, bookStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the BookingRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: BookingTasks.java with SEARCH_FLIGHTS, SELECT_SEAT, COMMIT_BOOKING constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9149 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (BookingAgent). The
  human-in-the-loop gate is an application-level workflow pause, not a second agent.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (BookingGuardrail) is the runtime mechanism that enforces
  phase order AND the confirmation precondition. Do NOT conditionally register tools per
  task — the guardrail is the gate.
- Task dependency is carried by typed task results: searchStep writes FlightOfferSet onto
  the entity, selectStep reads it and builds the SELECT task's instruction context from it,
  bookStep reads SeatSelection. The agent itself is stateless across phases.
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
