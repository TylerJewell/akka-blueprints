# SPEC — games-sales

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** VideoGamesSalesAssistant.
**One-line pitch:** A customer describes their preferences — genre, platform, budget, or a specific title — and one AI agent returns a structured `SalesRecommendation`: a ranked list of game suggestions with a per-title rationale, a confidence score, and an upsell opportunity, all validated by a guardrail before the response reaches the UI.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the sales-marketing domain. One `SalesAssistantAgent` (AutonomousAgent) handles the full recommendation decision; surrounding components prepare catalog context and audit the output. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** runs on every turn of `SalesAssistantAgent`. It checks that the candidate response is a valid `SalesRecommendation`: all required fields present, every `suggestions[].catalogId` exists in the seeded game catalog, every `confidenceScore` is between 0.0 and 1.0 inclusive, and the `suggestions` list is non-empty. A malformed response triggers a retry inside the same task, within the agent's iteration budget. No invalid recommendation reaches `SessionEntity`.

The blueprint shows that a single-agent pattern with customer-facing output warrants at minimum a structural guardrail — a bad recommendation seen by a customer is harder to retract than a bad internal verdict.

## 3. User-facing flows

The user opens the App UI tab.

1. The customer types a query into the **What are you looking for?** textarea (e.g., "I want a co-op action game for PS5 under $60") or picks one of three seeded example queries from the dropdown.
2. The customer optionally fills in **Platform preference**, **Genre preference**, and **Max budget** fields. These are encoded into the agent's task context, not the instructions, so the system can vary them per session without changing the prompt template.
3. The customer clicks **Get recommendations**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
4. The session card appears in the live list with status `QUERYING`. Within ~10–30 s, the agent returns. The card transitions to `RECOMMENDED`. The recommendation panel on the right shows: a ranked list of 1–5 suggested games, each with a title, platform, price, confidence score bar, rationale sentence, and an upsell chip (e.g., "Also consider the season pass").
5. The customer can click **Ask a follow-up** to submit a refinement query within the same session. The follow-up inherits the prior recommendation as context — the agent's task receives the prior `SalesRecommendation` as a follow-up attachment.
6. The customer can submit a fresh query, opening a new session; the live list keeps session history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SalesEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create, list, get, follow-up, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `RecommendationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: querying → recommended → follow-up → done. Source of truth. | `SalesEndpoint`, `SessionWorkflow` | `RecommendationView` |
| `SessionWorkflow` | `Workflow` | One workflow per session query. Steps: `queryStep` → `followUpStep` (optional). | started by `SalesEndpoint` on session create | `SalesAssistantAgent`, `SessionEntity` |
| `SalesAssistantAgent` | `AutonomousAgent` | The single decision-making LLM. Receives the customer query and preferences as task instructions; the optional prior recommendation as a task attachment. Returns `SalesRecommendation`. | invoked by `SessionWorkflow` | returns recommendation |
| `RecommendationGuardrail` | guardrail class | `before-agent-response` hook wired on `SalesAssistantAgent`. Validates `SalesRecommendation` structure and catalog refs. | `SalesAssistantAgent` | pass-through or rejection |
| `RecommendationView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SalesEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record GamePreferences(
    String query,
    Optional<String> platform,
    Optional<String> genre,
    Optional<Integer> maxBudgetCents,
    String sessionId,
    String customerId,
    Instant submittedAt
) {}

record GameSuggestion(
    String catalogId,
    String title,
    String platform,
    int priceCents,
    double confidenceScore,  // 0.0–1.0
    String rationale,
    Optional<String> upsellNote
) {}

record SalesRecommendation(
    List<GameSuggestion> suggestions,
    String summary,
    Instant decidedAt
) {}

record FollowUpQuery(
    String sessionId,
    String refinementText,
    Instant submittedAt
) {}

record Session(
    String sessionId,
    Optional<GamePreferences> preferences,
    Optional<SalesRecommendation> recommendation,
    Optional<SalesRecommendation> followUpRecommendation,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    QUERYING, RECOMMENDED, FOLLOW_UP_QUERYING, FOLLOW_UP_RECOMMENDED, FAILED
}
```

Events on `SessionEntity`: `SessionStarted`, `RecommendationRecorded`, `FollowUpStarted`, `FollowUpRecorded`, `SessionFailed`.

Every nullable lifecycle field on the `Session` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ query, platform?, genre?, maxBudgetCents?, customerId }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `POST /api/sessions/{id}/follow-up` — body `{ refinementText }` → `{ sessionId }` (same session, follow-up recommendation queued).
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Video Games Sales Assistant</title>`.

The App UI tab is a two-column layout: a left rail with a query submission panel and a live list of sessions (status pill + age + customer id), and a right pane with the selected session's detail — submitted preferences, ranked suggestions list, confidence score bars, rationale sentences, and upsell chips.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `SalesAssistantAgent`. Asserts the candidate response is parseable into `SalesRecommendation`, `suggestions` is non-empty, every `suggestions[].catalogId` appears in the seeded game catalog (loaded at startup from `src/main/resources/sample-events/game-catalog.jsonl`), every `confidenceScore` is in `[0.0, 1.0]`, and the `summary` field is non-empty. On failure returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `SalesAssistantAgent` → `prompts/sales-assistant.md`. The single decision-making LLM. System prompt instructs it to read the customer's query and preferences, consult its catalog context, and return a ranked `SalesRecommendation` with up to 5 suggestions.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer submits a PS5 action query; within 30 s the recommendation appears with 1–5 suggestions, each with a confidence score and rationale.
2. **J2** — The agent's first response is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed recommendation; the UI never displays the malformed response.
3. **J3** — A follow-up query in the same session produces a refined recommendation that references the prior one.
4. **J4** — A recommendation whose `suggestions[].catalogId` does not match any entry in the game catalog is rejected by the guardrail; the session ends in FAILED after exhausting retries.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named videogamessalesassistant demonstrating the single-agent x sales-marketing
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-sales-marketing-games-sales. Java package
io.akka.samples.videogamessalesassistant. Akka 3.6.0. HTTP port 9801.

Components to wire (exactly):

- 1 AutonomousAgent SalesAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/sales-assistant.md>) and
  .capability(TaskAcceptance.of(RECOMMEND_GAMES).maxIterationsPerTask(3)). The task receives
  the customer query and preferences as its instruction text; an optional prior recommendation
  as a task attachment (for follow-up queries) named "prior-recommendation.json". Output:
  SalesRecommendation{suggestions: List<GameSuggestion>, summary: String, decidedAt: Instant}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries the response within its 3-iteration budget.

- 1 Workflow SessionWorkflow per sessionId with two steps:
  * queryStep — emits SessionStarted, then calls componentClient.forAutonomousAgent(
    SalesAssistantAgent.class, "agent-" + sessionId).runSingleTask(
      TaskDef.instructions(formatQuery(session.preferences))
        [.attachment("prior-recommendation.json", priorJson) if follow-up]
    ) — returns a taskId, then forTask(taskId).result(RECOMMEND_GAMES) to fetch the
    recommendation. On success calls SessionEntity.recordRecommendation(recommendation).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(SessionWorkflow::error).
  * followUpStep — same pattern but fetches the session's existing recommendation as the
    attachment. WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(SessionWorkflow::error). error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  preferences: Optional<GamePreferences>, recommendation: Optional<SalesRecommendation>,
  followUpRecommendation: Optional<SalesRecommendation>, status: SessionStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. SessionStatus enum: QUERYING,
  RECOMMENDED, FOLLOW_UP_QUERYING, FOLLOW_UP_RECOMMENDED, FAILED. Events: SessionStarted{preferences},
  RecommendationRecorded{recommendation}, FollowUpStarted{refinementText},
  FollowUpRecorded{followUpRecommendation}, SessionFailed{reason}. Commands: startSession,
  recordRecommendation, startFollowUp, recordFollowUp, fail, getSession. emptyState()
  returns Session.initial("") with no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...)  inside
  the event-applier.

- 1 View RecommendationView with row type SessionRow (mirrors Session). Table updater
  consumes SessionEntity events. ONE query getAllSessions: SELECT * AS sessions FROM
  session_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * SalesEndpoint at /api with POST /sessions (body {query, platform?, genre?,
    maxBudgetCents?, customerId}; mints sessionId; calls SessionEntity.startSession; returns
    {sessionId}), POST /sessions/{id}/follow-up (body {refinementText}; calls
    SessionEntity.startFollowUp; re-runs workflow), GET /sessions (list from getAllSessions,
    sorted newest-first), GET /sessions/{id} (one row), GET /sessions/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SessionTasks.java declaring one Task<R> constant: RECOMMEND_GAMES = Task.name("Recommend
  games").description("Read the customer query and preferences and produce a
  SalesRecommendation with up to 5 ranked suggestions").resultConformsTo(
  SalesRecommendation.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records GamePreferences, GameSuggestion, SalesRecommendation, FollowUpQuery,
  Session, SessionStatus.

- RecommendationGuardrail.java implementing the before-agent-response hook. Loads the game
  catalog from src/main/resources/sample-events/game-catalog.jsonl at startup (a Set<String>
  of catalogId values). On each candidate SalesRecommendation: (1) parses the response into
  SalesRecommendation, (2) asserts suggestions is non-empty, (3) asserts every
  suggestions[].catalogId is in the loaded catalog set, (4) asserts every
  suggestions[].confidenceScore is in [0.0, 1.0], (5) asserts summary is non-empty. On any
  failure returns Guardrail.reject(<structured-error>) naming the failed check; the agent
  loop retries.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9801 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The SalesAssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/game-catalog.jsonl with 20 seeded game catalog entries.
  Each entry: {catalogId, title, platform, genre, priceCents, releaseYear, rating}. Covers
  PS5, Xbox Series X, Nintendo Switch, and PC. Genres: action, RPG, sports, puzzle, co-op.
  Examples: "elden-ring-ps5", "mario-kart-8-switch", "halo-infinite-xbox", "stardew-valley-pc".

- src/main/resources/sample-events/seed-queries.jsonl with 3 seeded customer queries:
  { "query": "I want a co-op action game for two players on PS5 under $70",
    "platform": "PS5", "genre": "action", "maxBudgetCents": 7000, "customerId": "cust-001" }
  { "query": "Best RPG for Nintendo Switch this year, any price",
    "platform": "Nintendo Switch", "genre": "RPG", "customerId": "cust-002" }
  { "query": "Something fun for a 10-year-old, Xbox Series X, puzzle or platformer",
    "platform": "Xbox Series X", "genre": "puzzle", "maxBudgetCents": 5000, "customerId": "cust-003" }

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with decisions.authority_level = recommend-only (the
  agent suggests, the customer decides), oversight.human_in_loop = false (automated
  customer-facing flow), failure.failure_modes including "hallucinated-catalog-entry",
  "irrelevant-suggestion", "out-of-range-confidence-score", "upsell-mismatch"; deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/sales-assistant.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Video Games Sales Assistant",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  query submission panel + live list of session cards; right = selected-session detail with
  preferences summary, ranked suggestions list with confidence bars, rationale, upsell chips,
  and follow-up query input). Browser title exactly:
  <title>Akka Sample: Video Games Sales Assistant</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    recommend-games.json — 8 SalesRecommendation entries covering varied suggestion counts
    (1–5) and confidence scores. Each entry has a summary and a suggestions array, where
    each GameSuggestion has a catalogId from the seeded game-catalog.jsonl, a non-empty
    rationale, and a confidenceScore between 0.1 and 1.0. Plus 2 deliberately MALFORMED
    entries: one with a catalogId that is not in the seeded catalog ("fake-game-xyz"), one
    with a confidenceScore of 1.5. The guardrail rejects both, exercising the retry path.
    The mock selects a malformed entry on the FIRST iteration of every 3rd session (modulo
    seed) so J2 is reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SalesAssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SessionTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (queryStep
  60s, followUpStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the Session row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: SessionTasks.java with RECOMMEND_GAMES = Task.name(...).description(...)
  .resultConformsTo(SalesRecommendation.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9801 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (SalesAssistantAgent).
  No additional LLM calls are made anywhere else in the system.
- The optional prior recommendation is passed as a Task ATTACHMENT for follow-up queries,
  not inlined into the instruction text. Verify the generated followUpStep uses
  TaskDef.attachment(...) and not string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
