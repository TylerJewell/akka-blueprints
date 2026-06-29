# SPEC — gemma-food-tour-guide

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GemmaFoodTourGuide.
**One-line pitch:** A traveler submits their dietary preferences, trip duration, and a target city; one AI agent reads a city food profile (passed as a task attachment, never as inline prompt text) and returns a structured, day-by-day culinary tour itinerary — named venues, meal slots, cultural notes, and a ranked stop list.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the planning-travel domain. One `FoodTourAgent` (AutonomousAgent) owns the full itinerary decision; the surrounding components prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **preference normalizer** runs inside a Consumer between the raw request submission and the agent call — traveler preferences containing personal health or dietary detail are reduced to category labels before the model processes them.
- A **before-agent-response guardrail** validates the agent's itinerary on every turn: well-formed JSON, every stop references a valid meal slot, every day index is within the trip duration, and the stop list covers at least one entry per requested dietary category. A malformed itinerary triggers a retry inside the same task.
- An **on-decision evaluator** runs immediately after each `ItineraryRecorded` event, scoring the itinerary for coverage quality (does every requested dietary category appear at least once? does the mix of venue types vary across days? are stop descriptions non-trivial?).

The blueprint shows that the single-agent pattern does not mean "ungoverned" — three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in the **Preferences** form: city (dropdown from seeded list — Tokyo, Barcelona, Mexico City), trip duration (1–7 days), dietary styles (checkboxes: vegetarian, vegan, pescatarian, omnivore, gluten-free, halal, kosher), budget tier (street-food / mid-range / fine-dining), and a free-text notes field.
2. The user clicks **Plan my tour**. The UI POSTs to `/api/tours` and receives a `tourId`.
3. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `PREFERENCES_VALIDATED` — the normalized category labels are visible in the card detail.
4. Within ~10–30 s, the workflow's `generateStep` completes. The card transitions to `GENERATING` then `ITINERARY_RECORDED`. The itinerary appears: a day-by-day breakdown, each day showing breakfast, lunch, dinner, and optional snack/market stops, with venue name, neighborhood, cultural context, and a one-line description.
5. Within ~1 s of the itinerary, the `evalStep` finishes. The card shows a **coverage score** chip (1–5) plus a one-line rationale.
6. The user can plan another tour; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TourEndpoint` | `HttpEndpoint` | `/api/tours/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TourRequestEntity`, `TourView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TourRequestEntity` | `EventSourcedEntity` | Per-tour lifecycle: requested → validated → generating → itinerary → scored. Source of truth. | `TourEndpoint`, `PreferenceValidator`, `TourWorkflow` | `TourView` |
| `PreferenceValidator` | `Consumer` | Subscribes to `TourRequested` events; normalizes free-text dietary notes to category labels; calls `TourRequestEntity.attachValidated`. | `TourRequestEntity` events | `TourRequestEntity` |
| `TourWorkflow` | `Workflow` | One workflow per tour request. Steps: `awaitValidatedStep` → `generateStep` → `evalStep`. | started by `PreferenceValidator` once validated event lands | `FoodTourAgent`, `TourRequestEntity` |
| `FoodTourAgent` | `AutonomousAgent` | The one decision-making LLM. Receives traveler preferences in the task definition and the city profile as a task attachment; returns `TourItinerary`. | invoked by `TourWorkflow` | returns itinerary |
| `TourView` | `View` | Read model: one row per tour request for the UI. | `TourRequestEntity` events | `TourEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TourPreferences(
    String city,
    int durationDays,
    List<String> dietaryStyles,      // e.g. ["vegetarian", "gluten-free"]
    BudgetTier budgetTier,
    String notes,                    // free-text; normalised before LLM
    String requestedBy,
    Instant requestedAt
) {}
enum BudgetTier { STREET_FOOD, MID_RANGE, FINE_DINING }

record ValidatedPreferences(
    String city,
    int durationDays,
    List<String> normalizedCategories, // canonical category labels
    BudgetTier budgetTier
) {}

record VenueStop(
    String venueId,
    String venueName,
    String neighborhood,
    MealSlot mealSlot,
    String dietaryCategory,
    String culturalNote,
    String description
) {}
enum MealSlot { BREAKFAST, LUNCH, DINNER, SNACK, MARKET }

record DayPlan(
    int dayIndex,                    // 1-based
    List<VenueStop> stops
) {}

record TourItinerary(
    ItineraryDecision decision,
    String summary,
    List<DayPlan> days,
    Instant generatedAt
) {}
enum ItineraryDecision { FULL_PLAN, PARTIAL_PLAN, NEEDS_MORE_INFO }

record CoverageResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record TourRequest(
    String tourId,
    Optional<TourPreferences> preferences,
    Optional<ValidatedPreferences> validated,
    Optional<TourItinerary> itinerary,
    Optional<CoverageResult> coverage,
    TourStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TourStatus {
    REQUESTED, PREFERENCES_VALIDATED, GENERATING, ITINERARY_RECORDED, SCORED, FAILED
}
```

Events on `TourRequestEntity`: `TourRequested`, `PreferencesValidated`, `GenerationStarted`, `ItineraryRecorded`, `CoverageScored`, `TourFailed`.

Every nullable lifecycle field on the `TourRequest` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tours` — body `{ city, durationDays, dietaryStyles: [String], budgetTier, notes, requestedBy }` → `{ tourId }`.
- `GET /api/tours` — list all tour requests, newest-first.
- `GET /api/tours/{id}` — one tour request.
- `GET /api/tours/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Gemma Food Tour Guide</title>`.

The App UI tab is a two-column layout: a left rail with the live list of tour requests (status pill + decision badge + age) and a right pane with the selected request's detail — validated preferences, day-by-day itinerary with venue cards per meal slot, and coverage score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `FoodTourAgent`. Asserts the candidate response is well-formed `TourItinerary` JSON, every `stops[].mealSlot` is in `{BREAKFAST, LUNCH, DINNER, SNACK, MARKET}`, every `dayIndex` is between 1 and the requested `durationDays`, and at least one stop per requested dietary category appears in the itinerary. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `ItineraryRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that every requested `dietaryStyle` is covered by at least one stop, that venue descriptions are non-trivial (length > 20 chars), and that no single day has all stops in the same `mealSlot`. Emits `CoverageScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `FoodTourAgent` → `prompts/food-tour-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached city profile, walk every requested day, and return one `DayPlan` per day with stops covering the traveler's dietary styles and budget tier.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a 3-day Tokyo tour request with vegetarian + street-food preferences; within 30 s the itinerary appears with at least 3 stops per day and a coverage score chip.
2. **J2** — The agent's first response on a request is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed itinerary; the UI never displays the malformed response.
3. **J3** — A request whose stops all share the same `mealSlot` receives a coverage score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A preferences note field containing "I have celiac disease and take metformin" is normalized to `["gluten-free"]` category label; the LLM call log shows only the normalized form.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named gemma-food-tour-guide demonstrating the single-agent × planning-travel
cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact single-agent-planning-travel-food-tour-guide.
Java package io.akka.samples.gemmafoodtourguide. Akka 3.6.0. HTTP port 9348.

Components to wire (exactly):

- 1 AutonomousAgent FoodTourAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/food-tour-agent.md>) and
  .capability(TaskAcceptance.of(PLAN_FOOD_TOUR).maxIterationsPerTask(3)). The task receives
  traveler preferences in the task definition and the city profile as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: TourItinerary{decision: ItineraryDecision, summary: String,
  days: List<DayPlan>, generatedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow TourWorkflow per tourId with three steps:
  * awaitValidatedStep — polls TourRequestEntity.getTourRequest every 1s; on
    request.validated().isPresent() advances to generateStep.
    WorkflowSettings.stepTimeout 15s.
  * generateStep — emits GenerationStarted, then calls componentClient.forAutonomousAgent(
    FoodTourAgent.class, "tour-agent-" + tourId).runSingleTask(
      TaskDef.instructions(formatPreferences(request.validated))
        .attachment("city-profile.txt",
          loadCityProfile(request.validated.city()).getBytes())
    ) — returns a taskId, then forTask(taskId).result(PLAN_FOOD_TOUR) to fetch the itinerary.
    On success calls TourRequestEntity.recordItinerary(itinerary).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(TourWorkflow::error).
  * evalStep — runs a deterministic rule-based CoverageScorer (NOT an LLM call) over the
    recorded itinerary: checks that every requested dietaryStyle has at least one matching
    stop, that no day is single-mealSlot-only, and that all stop descriptions exceed 20
    characters. Emits CoverageScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TourRequestEntity (one per tourId). State TourRequest{tourId: String,
  preferences: Optional<TourPreferences>, validated: Optional<ValidatedPreferences>,
  itinerary: Optional<TourItinerary>, coverage: Optional<CoverageResult>,
  status: TourStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  TourStatus enum: REQUESTED, PREFERENCES_VALIDATED, GENERATING, ITINERARY_RECORDED,
  SCORED, FAILED. Events: TourRequested{preferences}, PreferencesValidated{validated},
  GenerationStarted{}, ItineraryRecorded{itinerary}, CoverageScored{coverage},
  TourFailed{reason}. Commands: submit, attachValidated, markGenerating, recordItinerary,
  recordCoverage, fail, getTourRequest. emptyState() returns TourRequest.initial("") with
  no commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer PreferenceValidator subscribed to TourRequestEntity events; on TourRequested
  normalizes the free-text notes field by mapping health/medical terms to dietary category
  labels (e.g., "celiac" → "gluten-free", "lactose intolerant" → "dairy-free"), deduplicates
  against the explicit dietaryStyles list, and builds ValidatedPreferences. Calls
  TourRequestEntity.attachValidated(validated). After attachValidated lands, the same
  Consumer starts a TourWorkflow with id = "tour-" + tourId.

- 1 View TourView with row type TourRow (mirrors TourRequest). Table updater consumes
  TourRequestEntity events. ONE query getAllTours: SELECT * AS tours FROM tour_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * TourEndpoint at /api with POST /tours (body
    {city, durationDays, dietaryStyles: [String], budgetTier, notes, requestedBy};
    mints tourId; calls TourRequestEntity.submit; returns {tourId}), GET /tours
    (list from getAllTours, sorted newest-first), GET /tours/{id} (one row), GET
    /tours/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- TourTasks.java declaring one Task<R> constant: PLAN_FOOD_TOUR = Task.name("Plan food tour")
  .description("Read the city profile and traveler preferences and produce a TourItinerary
  covering every requested dietary style").resultConformsTo(TourItinerary.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TourPreferences, ValidatedPreferences, VenueStop, MealSlot, DayPlan,
  TourItinerary, ItineraryDecision, CoverageResult, TourRequest, TourStatus, BudgetTier.

- ItineraryGuardrail.java implementing the before-agent-response hook. Reads the candidate
  TourItinerary from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: TourItinerary and the
  ValidatedPreferences. Outputs: CoverageResult. Scoring rubric documented in Javadoc
  on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9348 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/city-profiles.jsonl with 3 seeded city profiles:
  Tokyo (neighborhoods: Shibuya, Tsukiji, Yanaka, Shimokitazawa; cuisine types: ramen,
  sushi, tempura, izakaya, depachika), Barcelona (El Born, La Barceloneta, Gràcia, Poble Sec;
  cuisine types: tapas, seafood, pintxos, vermouth bars, market stalls), Mexico City
  (Roma Norte, Coyoacán, La Merced, Polanco; cuisine types: tacos, tlayudas, pozole,
  antojitos, mezcalerías). Each profile includes 8–12 named venue entries with neighborhood,
  dietary tags, budget tier, and a 2-sentence description.

- src/main/resources/sample-events/seed-requests.jsonl with 3 paired example requests:
  a 3-day Tokyo trip (vegetarian + street-food), a 5-day Barcelona trip (pescatarian +
  mid-range), and a 2-day Mexico City trip (omnivore + fine-dining). Each contains
  a free-text notes value that the PreferenceValidator must normalize.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false (no PII in
  venue data), personal_health_notes_normalized_before_llm = true,
  decisions.authority_level = recommend-only (the itinerary is advisory; the traveler
  decides where to go), oversight.human_in_loop = true, failure.failure_modes including
  "hallucinated-venue", "missing-dietary-coverage", "day-index-out-of-range",
  "description-too-terse"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/food-tour-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Gemma Food Tour Guide", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of tour request cards; right = selected-request detail with validated
  preferences, day-by-day itinerary, venue cards per meal slot, and coverage score chip).
  Browser title exactly: <title>Akka Sample: Gemma Food Tour Guide</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java. Per-task mock-response shapes:
    plan-food-tour.json — 8 TourItinerary entries covering the three ItineraryDecision
      values and all three seeded cities. Each entry has a summary and a `days` array
      with one DayPlan per requested durationDays. Each DayPlan has 3–5 VenueStop entries
      across varied mealSlots. Each VenueStop has a non-empty venueName, neighborhood,
      description (> 20 chars), culturalNote, and a valid dietaryCategory. Plus 2
      deliberately MALFORMED entries (one with a dayIndex outside the requested duration;
      one with a mealSlot value outside the enum) — the guardrail blocks both, exercising
      the retry path.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FoodTourAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. TourTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (generateStep 60s,
  awaitValidatedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on TourRequest is Optional<T>.
- Lesson 7: TourTasks.java with PLAN_FOOD_TOUR mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9348 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: generated index.html includes mermaid CSS overrides and themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- Single-agent invariant: exactly ONE AutonomousAgent (FoodTourAgent). CoverageScorer is
  deterministic and makes NO LLM call.
- The city profile is passed as a Task ATTACHMENT, never inlined into instructions.
  Verify the generated generateStep uses TaskDef.attachment(...).
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
