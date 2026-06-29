# SPEC — restaurant-booking-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SK Restaurant Booking.
**One-line pitch:** A user describes a restaurant reservation they want; one AI agent collects all required parameters (restaurant, date, time, party size, contact details), presents a confirmation summary, and commits the booking via the Microsoft Graph Bookings API stub — only after the user explicitly approves.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the planning-travel domain. One `BookingAgent` (AutonomousAgent) carries the entire conversational collection and booking decision; the surrounding components manage lifecycle, confirmation routing, and state. Two governance mechanisms are wired around the agent:

- A **human-in-the-loop confirmation** pauses execution before the external write. The agent prepares a `BookingProposal` summarising restaurant name, date, time, party size, contact name, and total estimated cost; the workflow surfaces this to the user; only a user-supplied `CONFIRM` or `DECLINE` advances the booking.
- A **before-tool-call guardrail** validates every outbound booking write before the Graph Bookings stub call fires: the reservation date must be in the future, the party size must be between 1 and the restaurant's published maximum, the time slot must fall within operating hours, and the contact email must be syntactically valid. A rejected tool call is returned to the agent as a structured error; the agent corrects and retries.

The blueprint shows that even a conversational single-agent system carries independent governance layers. Neither mechanism is sufficient alone — the guardrail catches bad parameters the human confirmation cannot reason about; the confirmation gate catches cases where a valid payload is not what the user actually wanted.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a freeform reservation request into the **Request** textarea (e.g., "Book a table for four at Bella Napoli this Friday at 7 pm, for Alex Johnson, email alex@example.com") or picks one of three seeded examples from a dropdown.
2. The user clicks **Start booking**. The UI POSTs to `/api/bookings` and receives a `bookingId`.
3. The card appears in the live list in `INITIATED` state. Within ~5–20 s, the agent completes parameter collection and transitions the entity to `PENDING_CONFIRMATION`. A confirmation summary appears in the card detail: restaurant name, date, time, party size, contact name and email, and estimated covers charge.
4. The user clicks **Confirm** or **Decline** in the card detail. The UI POSTs to `/api/bookings/{id}/confirm` or `/api/bookings/{id}/decline`.
5. On confirm: the entity transitions to `COMMITTING`. Within ~2 s, the booking stub returns a `confirmationNumber`. The entity transitions to `CONFIRMED`. The card shows a green confirmation chip.
6. On decline: the entity transitions to `DECLINED` immediately. No external write occurs.
7. The user can start another booking; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BookingEndpoint` | `HttpEndpoint` | `/api/bookings/*` — start, list, get, confirm, decline, SSE; serves `/api/metadata/*`. | — | `BookingEntity`, `BookingView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BookingEntity` | `EventSourcedEntity` | Per-booking lifecycle: initiated → collecting → pending_confirmation → committing → confirmed / declined / failed. Source of truth. | `BookingEndpoint`, `BookingWorkflow` | `BookingView` |
| `ConfirmationConsumer` | `Consumer` | Subscribes to `ConfirmationRequested` events; routes the user's confirm/decline signal to the active `BookingWorkflow`. | `BookingEntity` events | `BookingWorkflow` |
| `BookingWorkflow` | `Workflow` | One workflow per booking. Steps: `collectStep` → `confirmStep` → `commitStep`. | started by `BookingEndpoint` on create | `BookingAgent`, `BookingEntity` |
| `BookingAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user's freeform request plus the restaurant catalogue as task instructions; returns a `BookingProposal`. Wired with `ReservationGuardrail`. | invoked by `BookingWorkflow` | returns proposal |
| `BookingView` | `View` | Read model: one row per booking for the UI. | `BookingEntity` events | `BookingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RestaurantInfo(
    String restaurantId,
    String name,
    String cuisine,
    String address,
    TimeRange operatingHours,   // { open: LocalTime, close: LocalTime }
    int maxPartySize,
    int estimatedCoverCharge    // in cents
) {}

record TimeRange(LocalTime open, LocalTime close) {}

record BookingRequest(
    String bookingId,
    String rawRequest,          // user's freeform text; preserved for audit
    String requestedBy,
    Instant initiatedAt
) {}

record BookingProposal(
    String restaurantId,
    String restaurantName,
    LocalDate date,
    LocalTime time,
    int partySize,
    String contactName,
    String contactEmail,
    int estimatedTotalCents,
    Instant proposedAt
) {}

record BookingConfirmation(
    String confirmationNumber,
    Instant confirmedAt
) {}

record Booking(
    String bookingId,
    Optional<BookingRequest> request,
    Optional<BookingProposal> proposal,
    Optional<BookingConfirmation> confirmation,
    BookingStatus status,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BookingStatus {
    INITIATED, COLLECTING, PENDING_CONFIRMATION, COMMITTING,
    CONFIRMED, DECLINED, FAILED
}
```

Events on `BookingEntity`: `BookingInitiated`, `CollectionStarted`, `ConfirmationRequested`, `BookingConfirmed`, `BookingDeclined`, `CommitStarted`, `BookingCompleted`, `BookingFailed`.

Every nullable lifecycle field on the `Booking` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/bookings` — body `{ rawRequest, requestedBy }` → `{ bookingId }`.
- `GET /api/bookings` — list all bookings, newest-first.
- `GET /api/bookings/{id}` — one booking.
- `POST /api/bookings/{id}/confirm` — user confirms the proposal; body `{}` → `204`.
- `POST /api/bookings/{id}/decline` — user declines; body `{}` → `204`.
- `GET /api/bookings/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SK Restaurant Booking</title>`.

The App UI tab is a two-column layout: a left rail with the live list of bookings (status pill + restaurant name + age) and a right pane with the selected booking's detail — the user's raw request, the proposal summary, and confirm/decline controls when in `PENDING_CONFIRMATION`, the confirmation number when `CONFIRMED`, and a failure reason when `FAILED`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — human-in-the-loop confirmation** (`hitl`, `application`): after `BookingAgent` returns a `BookingProposal`, the workflow enters `confirmStep` and emits `ConfirmationRequested`. Execution pauses until the user POSTs to `/api/bookings/{id}/confirm` or `/api/bookings/{id}/decline`. No external booking write happens without an explicit user confirmation. The proposal summary presented to the user covers all parameters the guardrail will later validate, giving the user a meaningful review opportunity.
- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`): registered on `BookingAgent` via the agent's guardrail-configuration block, bound to the `before-tool-call` hook. Fires before every call to the `createBooking` tool. Validates: (1) the reservation date is in the future, (2) party size is ≥ 1 and ≤ the restaurant's `maxPartySize`, (3) the requested time falls within the restaurant's `operatingHours`, and (4) `contactEmail` passes RFC 5322 syntax. On any failure returns a structured `invalid-tool-call` error to the agent loop; the agent corrects the field and retries. Passing calls proceed to the booking stub.

## 9. Agent prompts

- `BookingAgent` → `prompts/booking-agent.md`. The single decision-making LLM. System prompt instructs it to extract reservation parameters from the user's freeform request, look up the restaurant from the provided catalogue, and return a `BookingProposal`. It does not call the booking tool directly — the workflow owns the external write after confirmation.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded "Bella Napoli, 4 people" request; within 30 s a proposal appears, user confirms, and a confirmation number is returned.
2. **J2** — The agent proposes a booking for yesterday's date (mock LLM path); the `before-tool-call` guardrail rejects it; the agent corrects the date and retries; the confirmation eventually lands.
3. **J3** — User declines the confirmation; the entity transitions to `DECLINED` with no external write; the UI shows the declined card and the booking stub call count stays at zero.
4. **J4** — Booking stub returns a conflict error; the entity transitions to `FAILED`; the failure reason is visible in the UI card.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named skrestaurantbooking demonstrating the single-agent × planning-travel
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-planning-travel-restaurant-booking-agent. Java package
io.akka.samples.skrestaurantbooking. Akka 3.6.0. HTTP port 9279.

Components to wire (exactly):

- 1 AutonomousAgent BookingAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/booking-agent.md>) and
  .capability(TaskAcceptance.of(BOOK_RESTAURANT).maxIterationsPerTask(4)). The task receives
  the user's raw request plus the restaurant catalogue as its instruction text. Output:
  BookingProposal{restaurantId, restaurantName, date: LocalDate, time: LocalTime,
  partySize: int, contactName: String, contactEmail: String, estimatedTotalCents: int,
  proposedAt: Instant}. The agent is configured with a before-tool-call guardrail (see G1
  in eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries the tool call within its 4-iteration budget.

- 1 Workflow BookingWorkflow per bookingId with three steps:
  * collectStep — emits CollectionStarted, then calls componentClient.forAutonomousAgent(
    BookingAgent.class, "booker-" + bookingId).runSingleTask(
      TaskDef.instructions(formatRequest(booking.request, restaurantCatalogue))
    ) — returns a taskId, then forTask(taskId).result(BOOK_RESTAURANT) to fetch the proposal.
    On success emits ConfirmationRequested and calls BookingEntity.requestConfirmation(proposal).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(BookingWorkflow::error).
  * confirmStep — waits for ConfirmationConsumer to signal the workflow via
    BookingWorkflow.receiveConfirmation(decision: ConfirmationDecision). Decision enum:
    CONFIRMED, DECLINED. If DECLINED: calls BookingEntity.decline() and transitions to error
    step. If CONFIRMED: calls BookingEntity.startCommit() and advances to commitStep.
    WorkflowSettings.stepTimeout 300s (5 min user window).
  * commitStep — calls BookingStub.createBooking(proposal) — returns a BookingConfirmation
    carrying a confirmationNumber and confirmedAt. Calls BookingEntity.complete(confirmation).
    WorkflowSettings.stepTimeout 15s with defaultStepRecovery maxRetries(2)
    .failoverTo(BookingWorkflow::error). error step calls BookingEntity.fail(reason) and
    transitions to terminal.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BookingEntity (one per bookingId). State Booking{bookingId: String,
  request: Optional<BookingRequest>, proposal: Optional<BookingProposal>,
  confirmation: Optional<BookingConfirmation>, status: BookingStatus,
  failureReason: Optional<String>, createdAt: Instant, finishedAt: Optional<Instant>}.
  BookingStatus enum: INITIATED, COLLECTING, PENDING_CONFIRMATION, COMMITTING, CONFIRMED,
  DECLINED, FAILED. Events: BookingInitiated{request}, CollectionStarted{},
  ConfirmationRequested{proposal}, BookingConfirmed{}, CommitStarted{},
  BookingCompleted{confirmation}, BookingDeclined{}, BookingFailed{reason}.
  Commands: initiate, startCollection, requestConfirmation, confirm, decline, startCommit,
  complete, fail, getBooking. emptyState() returns Booking.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer ConfirmationConsumer subscribed to BookingEntity events; on
  ConfirmationRequested records the pending bookingId in an in-memory map keyed by bookingId.
  When BookingEndpoint routes a user confirm/decline POST, it calls
  ConfirmationConsumer.signal(bookingId, decision) which forwards the decision to the running
  BookingWorkflow via componentClient.forWorkflow(BookingWorkflow.class, bookingId)
  .call(BookingWorkflow::receiveConfirmation).

- 1 View BookingView with row type BookingRow (mirrors Booking minus request.rawRequest —
  the audit log keeps the raw; the view holds the proposal and confirmation). Table updater
  consumes BookingEntity events. ONE query getAllBookings: SELECT * AS bookings FROM
  booking_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * BookingEndpoint at /api with POST /bookings (body {rawRequest, requestedBy}; mints
    bookingId; calls BookingEntity.initiate; starts BookingWorkflow; returns {bookingId}),
    GET /bookings (list from getAllBookings, sorted newest-first), GET /bookings/{id} (one
    row), POST /bookings/{id}/confirm (calls ConfirmationConsumer.signal(CONFIRMED)),
    POST /bookings/{id}/decline (calls ConfirmationConsumer.signal(DECLINED)), GET
    /bookings/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- BookingTasks.java declaring one Task<R> constant: BOOK_RESTAURANT = Task.name("Book
  restaurant").description("Extract reservation parameters from the user request and
  return a BookingProposal for the selected restaurant").resultConformsTo(
  BookingProposal.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records RestaurantInfo, TimeRange, BookingRequest, BookingProposal,
  BookingConfirmation, Booking, BookingStatus.

- ReservationGuardrail.java implementing the before-tool-call hook. Reads the candidate
  tool-call parameters (restaurantId, date, time, partySize, contactEmail), loads the
  RestaurantInfo for the restaurantId from the in-memory catalogue, runs the four checks
  listed in eval-matrix.yaml G1, and either passes through or returns Guardrail.reject(
  <structured-error>) to force the agent loop to retry with corrected parameters.

- BookingStub.java — in-process stub simulating the Microsoft Graph Bookings API. On
  createBooking: validates the proposal, returns a BookingConfirmation with a synthetic
  confirmationNumber ("CONF-" + randomHex(6)). On a 20% pseudo-random chance (keyed by
  bookingId) returns a stub conflict error to exercise the FAILED path (journey J4).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9279 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/restaurants.json with 5 seeded restaurant entries:
  Bella Napoli (Italian, maxParty 8), The Golden Dragon (Chinese, maxParty 10), Le Petit
  Bistro (French, maxParty 6), Sakura Garden (Japanese, maxParty 12), Smoke & Ember (BBQ,
  maxParty 20). Each entry includes operating hours, address, and estimatedCoverCharge.

- src/main/resources/sample-events/booking-requests.jsonl with 3 seeded freeform request
  strings that exercise the three happy-path scenarios: party of 4 at Bella Napoli,
  business dinner for 2 at Le Petit Bistro, large group at Smoke & Ember.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true (contact name
  and email collected), decisions.authority_level = automated-action (the agent commits
  a real booking on behalf of the user), oversight.human_in_loop = true (explicit
  confirmation gate), failure.failure_modes including "wrong-restaurant", "wrong-date",
  "overbooking-conflict", "contact-data-misrouted"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/booking-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: SK Restaurant Booking", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of booking cards; right = selected-booking detail with raw request, proposal
  summary, confirm/decline controls when PENDING_CONFIRMATION, confirmation chip when
  CONFIRMED, failure reason when FAILED).
  Browser title exactly: <title>Akka Sample: SK Restaurant Booking</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct BookingProposal outputs (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. For BOOK_RESTAURANT: reads
  src/main/resources/mock-responses/book-restaurant.json, picks one entry pseudo-randomly
  per call (seedFor(bookingId)), and deserialises into BookingProposal.
- book-restaurant.json — 6 BookingProposal entries covering the five seeded restaurants
  with varied dates, times, and party sizes. Plus 2 deliberately INVALID entries:
  one with a date in the past (triggers the before-tool-call guardrail date check); one
  with partySize = 0 (triggers the party-size check). The mock selects an invalid entry
  on the first iteration of every 3rd booking (modulo seed) so J2 is reproducible.
- MockModelProvider.seedFor(bookingId) makes per-booking selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BookingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BookingTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent or waits on an external signal has an
  explicit stepTimeout (collectStep 60s, confirmStep 300s, commitStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the Booking row record is Optional<T>.
- Lesson 7: BookingTasks.java with BOOK_RESTAURANT = Task.name(...).description(...)
  .resultConformsTo(BookingProposal.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9279 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (BookingAgent). The booking
  stub is a deterministic in-process component — not an LLM. The confirm/decline decision
  is a human action, not an agent inference.
- The H1 human-in-the-loop gate is wired into the workflow's confirmStep. The external
  write in commitStep CANNOT proceed without a prior CONFIRMED signal from the user.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check after the agent returns.
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
