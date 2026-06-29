# SPEC — satellite-task-planner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Contract-Net Satellite Tasker.
**One-line pitch:** Submit an observation request; an auctioneer broadcasts it to a satellite fleet, collects feasibility-vetted bids, and awards the task to the highest-scoring winner.

## 2. What this blueprint demonstrates

The **contract-net** coordination pattern wired with Akka's first-party primitives: a Workflow broadcasts an observation request to all eligible satellites, each satellite submits a bid via an AutonomousAgent, a bid-feasibility guardrail blocks bids that contradict the satellite's ephemeris before the coordinator scores them, and the AuctionCoordinator awards the task to the winner. The blueprint also demonstrates a **periodic eval** that monitors awarded-task completion rate across the fleet and surfaces a fleet-health score.

## 3. User-facing flows

The user opens the App UI tab and submits an observation request via the form.

1. The system creates an `ObservationRequest` record in `OPEN` and starts an `ObservationAuction` workflow.
2. The workflow broadcasts the request to all registered satellites. Each satellite's `SatelliteBidder` AutonomousAgent evaluates the request against the satellite's current ephemeris window and returns a `Bid { satelliteId, windowStart, windowEnd, score, feasibilityNotes }`.
3. Before each bid is accepted, a bid-feasibility guardrail checks the bid's proposed window against the satellite's recorded ephemeris. Bids that contradict the ephemeris are rejected with `BID_INFEASIBLE`; the auction continues with the remaining bids.
4. After the bid-collection deadline (30 seconds) the `AuctionCoordinator` scores all accepted bids and awards the task to the highest-scoring satellite. The request moves to `AWARDED`.
5. If no feasible bids arrive before the deadline, the request moves to `UNALLOCATED`.
6. A `TaskCompletionSimulator` (TimedAction) marks a random `AWARDED` request as completed every 90 seconds so the completion-rate eval has data to measure.

An `ObservationRequestSimulator` (TimedAction) drips a sample request every 75 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AuctionCoordinator` | `AutonomousAgent` | Scores accepted bids and selects the winning satellite; runs award logic. | `ObservationAuction` | returns typed result to workflow |
| `SatelliteBidder` | `AutonomousAgent` | Evaluates an observation request against a satellite's current ephemeris window; returns a Bid. | `ObservationAuction` | — |
| `ObservationAuction` | `Workflow` | Broadcasts the request, collects bids within deadline, runs guardrail per bid, asks AuctionCoordinator to award. | `ObservationEndpoint`, `ObservationRequestConsumer` | `ObservationRequestEntity`, `SatelliteEntity` |
| `ObservationRequestEntity` | `EventSourcedEntity` | Holds the observation-request lifecycle (open → awarded / unallocated). | `ObservationAuction` | `ObservationView` |
| `SatelliteEntity` | `EventSourcedEntity` | Holds each satellite's operational state (orbit, windows, workload). | `ObservationAuction`, `SatelliteEndpoint` | `SatelliteView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted observation request for replay/audit. | `ObservationEndpoint`, `ObservationRequestSimulator` | `ObservationRequestConsumer` |
| `ObservationView` | `View` | List-of-requests read model. | `ObservationRequestEntity` events | `ObservationEndpoint` |
| `SatelliteView` | `View` | Fleet status read model. | `SatelliteEntity` events | `ObservationEndpoint` |
| `ObservationRequestConsumer` | `Consumer` | Listens to `RequestQueue` events; starts one `ObservationAuction` workflow per submission. | `RequestQueue` events | `ObservationAuction` |
| `ObservationRequestSimulator` | `TimedAction` | Drips a sample observation request every 75 s. | scheduler | `RequestQueue` |
| `TaskCompletionSimulator` | `TimedAction` | Marks one `AWARDED` request as completed every 90 s. | scheduler | `ObservationRequestEntity` |
| `FleetEvalSampler` | `TimedAction` | Every 10 minutes computes the fleet's awarded-task completion rate; emits a `FleetEvalScored` event. | scheduler | `ObservationRequestEntity` |
| `ObservationEndpoint` | `HttpEndpoint` | `/api/observations/*` and `/api/satellites/*` — submit, get, list, SSE. | — | `ObservationView`, `SatelliteView`, `RequestQueue`, `ObservationRequestEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ObservationRequest(String requestId, String target, String instrument,
                          Instant requestedAt, String requestedBy) {}

record EphemerisWindow(Instant windowStart, Instant windowEnd, String orbitType,
                       double elevationDegrees) {}

record Bid(String satelliteId, Instant windowStart, Instant windowEnd,
           double score, String feasibilityNotes) {}

record BidBundle(List<Bid> bids, Instant collectedAt) {}

record AwardDecision(String winningSatelliteId, Bid winningBid,
                     String rationale, Instant awardedAt) {}

record ObservationRequestState(
    String requestId,
    String target,
    String instrument,
    RequestStatus status,
    Optional<BidBundle> bids,
    Optional<AwardDecision> award,
    Optional<String> failureReason,
    Optional<Double> fleetCompletionRate,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

record SatelliteState(
    String satelliteId,
    String name,
    String orbitType,
    boolean operational,
    Optional<String> assignedRequestId,
    int completedCount,
    Instant registeredAt
) {}

enum RequestStatus { OPEN, BIDDING, AWARDED, UNALLOCATED, COMPLETED }
```

### Events (on `ObservationRequestEntity`)

`RequestOpened`, `BiddingStarted`, `BidReceived`, `BidRejected`, `TaskAwarded`, `TaskUnallocated`, `TaskCompleted`, `FleetEvalScored`.

### Events (on `SatelliteEntity`)

`SatelliteRegistered`, `SatelliteAssigned`, `SatelliteReleased`.

### Events (on `RequestQueue`)

`ObservationSubmitted { requestId, target, instrument, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/observations` — body `{ target, instrument }` → `{ requestId }`. Starts an auction workflow.
- `GET /api/observations` — list all observation requests. Optional `?status=OPEN|BIDDING|AWARDED|UNALLOCATED|COMPLETED`.
- `GET /api/observations/{id}` — one request.
- `GET /api/observations/sse` — server-sent events stream of every request change.
- `GET /api/satellites` — list all registered satellites with current state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Contract-Net Satellite Tasker"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an observation request, live list of requests with status pills, expand-row to see bids received, the winning bid, award rationale, and fleet completion rate.

Browser title: `<title>Akka Sample: Contract-Net Satellite Tasker</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — bid-feasibility guardrail** (`before-bid-acceptance` on `ObservationAuction`): checks each bid's proposed window against the satellite's recorded ephemeris before the bid is admitted to the scoring pool. Blocking. Infeasible bids are rejected with `BidRejected`.
- **E1 — fleet completion-rate eval** (`scheduled-eval`): `FleetEvalSampler` (TimedAction) runs every 10 minutes, computes the ratio of `COMPLETED` to `AWARDED` requests, and emits a `FleetEvalScored` event.

## 9. Agent prompts

- `AuctionCoordinator` → `prompts/auction-coordinator.md`. Scores accepted bids and selects the winning satellite.
- `SatelliteBidder` → `prompts/satellite-bidder.md`. Evaluates an observation request against an ephemeris window; returns a Bid.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an observation request; it progresses `OPEN → BIDDING → AWARDED` within 60 s; the App UI shows the winning bid and award rationale.
2. **J2** — A satellite submits a bid with an impossible window (elevation < 0°); the guardrail rejects the bid with `BID_INFEASIBLE` before scoring; the auction continues with remaining bids.
3. **J3** — All satellites are out-of-window; no feasible bids arrive before the deadline; the request enters `UNALLOCATED`.
4. **J4** — `TaskCompletionSimulator` marks an awarded request completed; `FleetEvalSampler` records the updated completion rate.
5. **J5** — `ObservationRequestSimulator` drips requests every 75 s; App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named satellite-task-planner demonstrating the
contract-net × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
contract-net-research-intel-satellite-task-planner.
Java package io.akka.samples.contractnetsatellitetasker. Akka 3.6.0.
HTTP port 9982.

Components to wire (exactly):
- 2 AutonomousAgents:
  * AuctionCoordinator — definition() with
    capability(TaskAcceptance.of(SCORE_BIDS).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(AWARD_TASK).maxIterationsPerTask(2)).
    System prompt loaded from prompts/auction-coordinator.md.
    Returns BidBundle{bids, collectedAt} for SCORE_BIDS and
    AwardDecision{winningSatelliteId, winningBid, rationale, awardedAt} for AWARD_TASK.
  * SatelliteBidder — capability(TaskAcceptance.of(BID).maxIterationsPerTask(2)).
    System prompt from prompts/satellite-bidder.md.
    Returns Bid{satelliteId, windowStart, windowEnd, score, feasibilityNotes}.

- 1 Workflow ObservationAuction with steps:
  openStep -> broadcastStep -> [parallel per satellite] bidStep ->
  collectStep -> awardStep -> emitStep.
  openStep calls ObservationRequestEntity.openRequest.
  broadcastStep reads SatelliteView.getOperationalSatellites and fans
  out one bidStep per satellite.
  Each bidStep calls forAutonomousAgent(SatelliteBidder.class, BID);
  stepTimeout 30s per bid step (Lesson 4). Each accepted bid passes a
  before-bid-acceptance guardrail that checks windowStart/windowEnd
  against the satellite's EphemerisWindow; failing bids call
  ObservationRequestEntity.rejectBid and are dropped from the pool.
  collectStep merges accepted bids into BidBundle; if empty, transitions
  to unallocatedStep that calls ObservationRequestEntity.unallocate.
  awardStep calls forAutonomousAgent(AuctionCoordinator.class, AWARD_TASK)
  with the BidBundle; stepTimeout 45s. On success calls
  ObservationRequestEntity.awardTask(winningSatelliteId, winningBid) and
  SatelliteEntity.assignSatellite(requestId).
  WorkflowSettings is nested inside Workflow — no import.

- 2 EventSourcedEntities:
  * ObservationRequestEntity holding state ObservationRequestState{
    requestId, target, instrument, RequestStatus, Optional<BidBundle> bids,
    Optional<AwardDecision> award, Optional<String> failureReason,
    Optional<Double> fleetCompletionRate, Instant createdAt,
    Optional<Instant> finishedAt}.
    RequestStatus enum: OPEN, BIDDING, AWARDED, UNALLOCATED, COMPLETED.
    Events: RequestOpened, BiddingStarted, BidReceived, BidRejected,
    TaskAwarded, TaskUnallocated, TaskCompleted, FleetEvalScored.
    Commands: openRequest, startBidding, receiveBid, rejectBid, awardTask,
    unallocate, completeTask, recordFleetEval, getRequest.
    emptyState() returns ObservationRequestState.initial("", null) with
    no commandContext() reference.
  * SatelliteEntity holding state SatelliteState{satelliteId, name,
    orbitType, operational, Optional<String> assignedRequestId,
    int completedCount, Instant registeredAt}.
    Events: SatelliteRegistered, SatelliteAssigned, SatelliteReleased.
    Commands: registerSatellite, assignSatellite, releaseSatellite, getSatellite.

- 1 EventSourcedEntity RequestQueue with command enqueueObservation(
  target, instrument, requestedBy) emitting
  ObservationSubmitted{requestId, target, instrument, requestedBy, submittedAt}.

- 2 Views:
  * ObservationView with row type ObservationRequestRow (mirrors
    ObservationRequestState minus heavy nested payloads; every nullable
    field is Optional<T>). Table updater consumes ObservationRequestEntity
    events. ONE query getAllRequests SELECT * AS requests FROM observation_view.
    No WHERE status filter — caller filters client-side.
  * SatelliteView with row type SatelliteRow (mirrors SatelliteState).
    Table updater consumes SatelliteEntity events.
    ONE query getOperationalSatellites SELECT * AS satellites FROM satellite_view.

- 1 Consumer ObservationRequestConsumer subscribed to RequestQueue events;
  on ObservationSubmitted starts an ObservationAuction with requestId
  as the workflow id.

- 3 TimedActions:
  * ObservationRequestSimulator — every 75s reads next line from
    src/main/resources/sample-events/observation-requests.jsonl and
    calls RequestQueue.enqueueObservation.
  * TaskCompletionSimulator — every 90s picks a random AWARDED request
    from ObservationView and calls ObservationRequestEntity.completeTask.
  * FleetEvalSampler — every 10 minutes queries ObservationView for
    all AWARDED and COMPLETED requests, computes ratio of COMPLETED to
    (AWARDED + COMPLETED), and calls ObservationRequestEntity.recordFleetEval
    on the most recent completed request, emitting FleetEvalScored.

- 2 HttpEndpoints:
  * ObservationEndpoint at /api with POST /observations, GET /observations,
    GET /observations/{id}, GET /observations/sse, GET /satellites, and
    the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- SatelliteTasks.java declaring three Task<R> constants: BID (Bid),
  SCORE_BIDS (BidBundle), AWARD_TASK (AwardDecision).
- Domain records ObservationRequest, EphemerisWindow, Bid, BidBundle,
  AwardDecision, ObservationRequestState, SatelliteState.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9982 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read
  from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/observation-requests.jsonl with 8
  canned observation-request lines (varying target, instrument).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 bid-feasibility
  guardrail before-bid-acceptance, E1 fleet completion-rate scheduled-eval)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling purpose and capabilities
  fields for the satellite-tasker domain; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/auction-coordinator.md, prompts/satellite-bidder.md loaded at
  agent startup as system prompts.
- README.md at the project root.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs: Overview, Architecture
  (4 mermaid diagrams + click-to-expand component table with syntax-highlighted
  Java snippets), Risk Survey, Eval Matrix, App UI. Browser title exactly:
  <title>Akka Sample: Contract-Net Satellite Tasker</title>. No subtitle
  on the Overview tab. Seed the satellite fleet with 4 satellites on startup
  (LEO-1 through LEO-4) via a TimedAction that fires once.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value in Claude session memory only.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class name.
  Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json:
    auction-coordinator.json — 4–6 AwardDecision entries, each naming a
      winning satelliteId (LEO-1 through LEO-4), a Bid with a valid window,
      a score 0.6–0.95, and a 2–3 sentence rationale.
    satellite-bidder.json — 4–6 Bid entries per satellite, each with a
      windowStart 30–90 minutes from now, windowEnd windowStart + 20 minutes,
      score 0.5–0.99, and short feasibilityNotes.
- MockModelProvider.seedFor(requestId) makes selection deterministic per
  request id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (30s per bid
  step, 45s award step); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion SatelliteTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9982 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from DOM, do not display:none them.
- Parallel bid fan-out uses CompletionStage zip over all satellite bid futures, NOT sequential
  step calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, simpler, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone, competitor brand names.
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
