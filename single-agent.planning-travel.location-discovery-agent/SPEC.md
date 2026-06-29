# SPEC — location-discovery-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Foursquare Location Agent.
**One-line pitch:** A user submits a natural-language search query and a GPS coordinate; one AI agent reads the place data (passed as a task attachment, never as inline prompt text) and returns a structured ranked recommendation — a list of nearby places scored by relevance, with a short rationale per place.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the planning-travel domain. One `LocationDiscoveryAgent` (AutonomousAgent) carries the entire recommendation decision; the surrounding components prepare its input and audit its output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw coordinate submission and the agent call — so the model never sees the user's exact GPS position, only a bounding-box token.
- A **before-agent-response guardrail** validates the agent's recommendation on every turn: well-formed JSON, every result references a real place id from the candidate set, every score is in the allowed range. A malformed recommendation triggers a retry inside the same task.
- An **on-decision-eval** runs immediately after each `RecommendationRecorded` event, scoring the recommendation for relevance quality (does every place have a rationale? is the score ordering internally consistent? are rationales non-generic?) as a deterministic rule-based check.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a search query into the **Search** field (e.g., "coffee shops with good wifi for remote work").
2. The user enters a GPS coordinate — latitude and longitude — in the **Location** fields, or picks one of three seeded city areas (Downtown SF, Midtown NYC, Central London).
3. The user selects a **radius** (500 m / 1 km / 2 km) and an optional **category filter** (All / Food & Drink / Retail / Arts & Entertainment / Outdoors).
4. The user clicks **Find places**. The UI POSTs to `/api/searches` and receives a `searchId`.
5. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `COORDINATE_SANITIZED` — the bounding-box token is visible in the card detail instead of the raw coordinates.
6. Within ~10–30 s, the workflow's `discoverStep` completes. The card transitions to `DISCOVERING` then `RECOMMENDATION_RECORDED`. The recommendation appears: a ranked list of places (name, category, distance, relevance score 1–10, one-sentence rationale), plus a summary paragraph.
7. Within ~1 s of the recommendation, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale.
8. The user can submit another search; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SearchEndpoint` | `HttpEndpoint` | `/api/searches/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `SearchEntity`, `SearchView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SearchEntity` | `EventSourcedEntity` | Per-search lifecycle: submitted → coordinate_sanitized → discovering → recommendation → eval. Source of truth. | `SearchEndpoint`, `CoordinateSanitizer`, `DiscoveryWorkflow` | `SearchView` |
| `CoordinateSanitizer` | `Consumer` | Subscribes to `SearchSubmitted` events; redacts exact GPS to bounding-box token; calls `SearchEntity.attachSanitized`. | `SearchEntity` events | `SearchEntity` |
| `DiscoveryWorkflow` | `Workflow` | One workflow per search. Steps: `awaitSanitizedStep` → `discoverStep` → `evalStep`. | started by `CoordinateSanitizer` once sanitized event lands | `LocationDiscoveryAgent`, `SearchEntity` |
| `LocationDiscoveryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the search query and category filter in the task definition; receives the candidate place dataset and bounding-box token as a task attachment; returns `PlaceRecommendation`. | invoked by `DiscoveryWorkflow` | returns recommendation |
| `SearchView` | `View` | Read model: one row per search for the UI. | `SearchEntity` events | `SearchEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PlaceCandidate(
    String placeId,
    String name,
    String category,
    double distanceMeters,
    double latitude,
    double longitude,
    String address,
    Optional<Double> rating,
    Optional<Integer> reviewCount
) {}

record SearchRequest(
    String searchId,
    String query,
    double rawLatitude,
    double rawLongitude,
    int radiusMeters,
    Optional<String> categoryFilter,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedCoordinate(
    String boundingBoxToken,     // e.g. "bbox:37.77-37.79:-122.42--122.40"
    List<PlaceCandidate> candidatePlaces
) {}

record PlaceResult(
    String placeId,
    String name,
    String category,
    double distanceMeters,
    int relevanceScore,          // 1..10
    String rationale
) {}

record PlaceRecommendation(
    String summary,
    List<PlaceResult> places,    // ordered by relevanceScore descending
    Instant decidedAt
) {}

record EvalResult(
    int score,                   // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Search(
    String searchId,
    Optional<SearchRequest> request,
    Optional<SanitizedCoordinate> sanitized,
    Optional<PlaceRecommendation> recommendation,
    Optional<EvalResult> eval,
    SearchStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SearchStatus {
    SUBMITTED, COORDINATE_SANITIZED, DISCOVERING, RECOMMENDATION_RECORDED, EVALUATED, FAILED
}
```

Events on `SearchEntity`: `SearchSubmitted`, `CoordinateSanitized`, `DiscoveryStarted`, `RecommendationRecorded`, `EvaluationScored`, `SearchFailed`.

Every nullable lifecycle field on the `Search` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/searches` — body `{ query, rawLatitude, rawLongitude, radiusMeters, categoryFilter, submittedBy }` → `{ searchId }`.
- `GET /api/searches` — list all searches, newest-first.
- `GET /api/searches/{id}` — one search.
- `GET /api/searches/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Foursquare Location Agent</title>`.

The App UI tab is a two-column layout: a left rail with the search submission form plus the live list of submitted searches (status pill + age + query snippet) and a right pane with the selected search's detail — sanitized coordinate token, candidate place count, recommendation list with relevance scores, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `CoordinateSanitizer` Consumer): redacts the user's exact GPS latitude/longitude to a coarse bounding-box token before any LLM call. The raw coordinates are preserved on the entity for audit but the model only receives the token. This prevents the model from learning, memorising, or echoing a user's precise physical location.
- **G1 — before-agent-response guardrail**: runs on every turn of `LocationDiscoveryAgent`. Asserts the candidate response is well-formed `PlaceRecommendation` JSON, every `places[].placeId` appears in the submitted `SanitizedCoordinate.candidatePlaces` list, every `relevanceScore` is in the range 1–10, and the `places` list is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `RecommendationRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that every `PlaceResult` has a non-empty rationale, that the rationale is not a generic filler string (checked against a banned-phrase list), and that `relevanceScore` values are not all identical (a collapsed ranking loses 1 point). Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `LocationDiscoveryAgent` → `prompts/location-discovery.md`. The single decision-making LLM. System prompt instructs it to read the attached candidate place list, consider the query and category filter, and return a ranked `PlaceRecommendation` with a relevance score and rationale per place.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a coffee-shop query for Downtown SF; within 30 s the recommendation appears with a ranked place list and an eval score chip.
2. **J2** — The agent's first response on a search is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed recommendation; the UI never displays the malformed response.
3. **J3** — A recommendation whose rationale strings are all generic filler receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A search containing precise GPS coordinates is submitted; the LLM call log shows only the bounding-box token; the entity's `request.rawLatitude` and `request.rawLongitude` retain the raw values for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named location-discovery-agent demonstrating the single-agent × planning-travel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-planning-travel-location-discovery-agent. Java package
io.akka.samples.foursquarelocationagent. Akka 3.6.0. HTTP port 9150.

Components to wire (exactly):

- 1 AutonomousAgent LocationDiscoveryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/location-discovery.md>) and
  .capability(TaskAcceptance.of(DISCOVER_PLACES).maxIterationsPerTask(3)). The task receives
  the search query and category filter as its instruction text and the candidate place dataset
  (serialised as JSON) as a task ATTACHMENT named "places.json"
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: PlaceRecommendation{summary: String, places: List<PlaceResult>,
  decidedAt: Instant}. The agent is configured with a before-agent-response guardrail (see G1
  in eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow DiscoveryWorkflow per searchId with three steps:
  * awaitSanitizedStep — polls SearchEntity.getSearch every 1s; on search.sanitized().isPresent()
    advances to discoverStep. WorkflowSettings.stepTimeout 15s.
  * discoverStep — emits DiscoveryStarted, then calls componentClient.forAutonomousAgent(
    LocationDiscoveryAgent.class, "discoverer-" + searchId).runSingleTask(
      TaskDef.instructions(formatQuery(search.request))
        .attachment("places.json",
          serializeCandidates(search.sanitized.candidatePlaces).getBytes())
    ) — returns a taskId, then forTask(taskId).result(DISCOVER_PLACES) to fetch the
    recommendation. On success calls SearchEntity.recordRecommendation(recommendation).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(DiscoveryWorkflow::error).
  * evalStep — runs a deterministic rule-based EvaluationScorer (NOT an LLM call) over the
    recorded recommendation: checks that every PlaceResult has a non-empty rationale, that
    rationale strings are not generic filler (checked against a banned-phrase list), and that
    relevanceScore values are not all identical. Emits EvaluationScored{score: 1-5, rationale:
    String}. WorkflowSettings.stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SearchEntity (one per searchId). State Search{searchId: String,
  request: Optional<SearchRequest>, sanitized: Optional<SanitizedCoordinate>,
  recommendation: Optional<PlaceRecommendation>, eval: Optional<EvalResult>, status:
  SearchStatus, createdAt: Instant, finishedAt: Optional<Instant>}. SearchStatus enum:
  SUBMITTED, COORDINATE_SANITIZED, DISCOVERING, RECOMMENDATION_RECORDED, EVALUATED, FAILED.
  Events: SearchSubmitted{request}, CoordinateSanitized{sanitized}, DiscoveryStarted{},
  RecommendationRecorded{recommendation}, EvaluationScored{eval}, SearchFailed{reason}.
  Commands: submit, attachSanitized, markDiscovering, recordRecommendation, recordEvaluation,
  fail, getSearch. emptyState() returns Search.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer CoordinateSanitizer subscribed to SearchEntity events; on SearchSubmitted redacts
  the exact rawLatitude/rawLongitude to a coarse bounding-box token (±0.01 degree cells,
  formatted as "bbox:<latMin>-<latMax>:<lonMin>-<lonMax>"), loads the in-process seeded place
  dataset, filters to candidates within the requested radiusMeters of the raw coordinate,
  builds SanitizedCoordinate{boundingBoxToken, candidatePlaces}, then calls
  SearchEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer starts
  a DiscoveryWorkflow with id = "discovery-" + searchId.

- 1 View SearchView with row type SearchRow (mirrors Search minus request.rawLatitude and
  request.rawLongitude — the audit log keeps the raw coordinates; the view holds the bounding-
  box token for the UI). Table updater consumes SearchEntity events. ONE query getAllSearches:
  SELECT * AS searches FROM search_view. No WHERE status filter — Akka cannot auto-index enum
  columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SearchEndpoint at /api with POST /searches (body {query, rawLatitude, rawLongitude,
    radiusMeters, categoryFilter, submittedBy}; mints searchId; calls SearchEntity.submit;
    returns {searchId}), GET /searches (list from getAllSearches, sorted newest-first), GET
    /searches/{id} (one row), GET /searches/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SearchTasks.java declaring one Task<R> constant: DISCOVER_PLACES = Task.name("Discover
  places").description("Read the attached candidate place list and produce a PlaceRecommendation
  ranked by relevance to the query").resultConformsTo(PlaceRecommendation.class). DO NOT skip
  this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PlaceCandidate, SearchRequest, SanitizedCoordinate, PlaceResult,
  PlaceRecommendation, EvalResult, Search, SearchStatus.

- RecommendationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  PlaceRecommendation from the LLM response, runs the three checks listed in eval-matrix.yaml
  G1, and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- EvaluationScorer.java — pure deterministic logic (no LLM). Inputs: PlaceRecommendation and
  the submitted SearchRequest. Outputs: EvalResult. Scoring rubric documented in Javadoc on the
  class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9150 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The LocationDiscoveryAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-places.jsonl with 30 seeded PlaceCandidate entries
  spread across three city clusters (Downtown SF, Midtown NYC, Central London). Each cluster
  has 10 places across at least 3 categories (Food & Drink, Retail, Arts & Entertainment).
  Each entry includes a placeId, name, category, approximate latitude/longitude, distance
  placeholder (computed at runtime), and a rating.

- src/main/resources/sample-events/seed-searches.jsonl with 3 seeded search queries with
  paired coordinates: (1) "coffee shops with good wifi" → Downtown SF coords, (2)
  "bookstores and independent shops" → Midtown NYC coords, (3) "outdoor parks and gardens" →
  Central London coords.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.location-fine-grained = true,
  coordinate_handled_by_sanitizer_before_llm = true,
  decisions.authority_level = recommend-only (the agent's ranking is advisory, not enforced),
  oversight.human_in_loop = true (a user reads the recommendation before acting on it),
  failure.failure_modes including "hallucinated-place", "stale-place-data",
  "location-leakage-via-llm", "ranking-collapse"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/location-discovery.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Foursquare Location Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  search form + live list of search cards; right = selected-search detail with bounding-box
  token, candidate place count, recommendation list with relevance scores, and eval-score chip).
  Browser title exactly: <title>Akka Sample: Foursquare Location Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(searchId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    discover-places.json — 8 PlaceRecommendation entries covering varied place lists. Each
    entry has a summary and a places array with 3–5 PlaceResult items. Each PlaceResult has
    a non-empty rationale and a relevanceScore between 1 and 10. Plus 2 deliberately MALFORMED
    entries (one with a placeId not in the submitted candidate list; one with a relevanceScore
    outside 1–10) — the guardrail blocks both, exercising the retry path. The mock selects a
    malformed entry on the FIRST iteration of every 3rd search (modulo seed) so J2 is
    reproducible.
- A MockModelProvider.seedFor(searchId) helper makes per-search selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. LocationDiscoveryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SearchTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (discoverStep
  60s, awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Search row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: SearchTasks.java with DISCOVER_PLACES = Task.name(...).description(...)
  .resultConformsTo(PlaceRecommendation.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9150 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the mermaid.initialize
  themeVariables block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (LocationDiscoveryAgent).
  The on-decision eval is rule-based (EvaluationScorer.java) and does NOT make an LLM call.
- The candidate place list is passed as a Task ATTACHMENT named "places.json", never inlined
  into the agent's instructions. Verify the generated discoverStep uses TaskDef.attachment(...)
  and not string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
