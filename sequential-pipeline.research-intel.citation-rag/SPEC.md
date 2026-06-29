# SPEC — citation-rag

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Citation Query Engine.
**One-line pitch:** A user submits a research query; one `CitationAgent` walks it through three task phases — **RETRIEVE** source passages, **ATTRIBUTE** each extracted claim to a passage, **COMPOSE** a grounded answer — with every claim in the final answer verifiably linked to a cited passage before release.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a research-intelligence domain. One `CitationAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RETRIEVE task's typed output becomes the ATTRIBUTE task's instruction context; the ATTRIBUTE task's typed output becomes the COMPOSE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- An **`after-llm-response` guardrail** runs after the COMPOSE task returns its `Answer` and before the answer is committed to the entity. It inspects every `Claim` in the answer and checks that each carries a non-null `citedPassageId` that resolves to a passage in the upstream `PassageSet`. A claim with no citation, or a citation pointing to a passage not in the collected set, is a citation violation. The guardrail rejects the answer as a whole, returns a structured `uncited-claim` error to the agent loop, and lets the agent retry the COMPOSE task within its 4-iteration budget. The same hook is the right cut: the LLM already produced output, but the output fails a verifiable post-generation contract before it becomes permanent.
- An **`on-decision-eval`** runs immediately after `AnswerComposed` lands, as `evalStep` inside the workflow. A deterministic, rule-based `CoverageScorer` (no LLM call — the eval is rule-based on purpose) checks that every claim in the answer is attributed, that every cited passage id resolves in the collected `PassageSet`, that the answer's claim count is non-zero, and that every cited passage contributes at least one claim (no decoration citations). Emits `CitationScored{score:1..5, rationale}`.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the after-response guardrail and the on-decision evaluator are complementary checks at two different cut points, and neither silently covers what the other misses.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **research query** into the input (or picks one of three seeded queries — `What are the main findings on transformer attention efficiency?`, `How do RAG systems handle conflicting sources?`, `What evaluation metrics are used for citation quality?`).
2. The user clicks **Run query**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RETRIEVING` — the workflow has started `retrieveStep` and the agent has been handed the RETRIEVE task.
4. Within ~10–20 s the card reaches `ATTRIBUTING` — the typed `PassageSet` is visible in the card detail (a small table of passages with source, text preview, and score). The agent's RETRIEVE task returned; the workflow recorded `PassagesRetrieved` and ran the ATTRIBUTE task.
5. Within ~10–20 s more the card reaches `COMPOSING`. The `ClaimSet` is visible (claim list with passage attributions and confidence).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `Answer` — query restatement, a grounded response body, and a per-claim citations table — plus a citation-coverage score chip (1–5) and a one-line rationale.
7. The user can submit another query; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: created → retrieving → retrieved → attributing → attributed → composing → composed → evaluated. Source of truth. | `QueryEndpoint`, `CitationQueryWorkflow` | `QueryView` |
| `CitationQueryWorkflow` | `Workflow` | One workflow per query. Steps: `retrieveStep` → `attributeStep` → `composeStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `QueryEndpoint` after `CREATED` | `CitationAgent`, `QueryEntity` |
| `CitationAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `CitationTasks.java`: `RETRIEVE_PASSAGES` → `PassageSet`, `ATTRIBUTE_CLAIMS` → `ClaimSet`, `COMPOSE_ANSWER` → `Answer`. Each task is registered with the phase-appropriate function tools. The after-llm-response guardrail (`CitationGuardrail`) is registered on the agent. | invoked by `CitationQueryWorkflow` | returns typed results |
| `RetrieveTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchPassages(query)` and `fetchPassage(passageId)`. Reads from `src/main/resources/sample-data/corpus/*.json` for deterministic offline output. | called from RETRIEVE task | returns `List<Passage>` |
| `AttributeTools` | function-tools class | Implements `extractClaims(passages)` and `linkClaims(claims, passages)`. Pure in-memory transformations that produce attribution linkages. | called from ATTRIBUTE task | returns `List<Claim>` |
| `ComposeTools` | function-tools class | Implements `draftAnswer(claims, query)` and `formatCitations(claims, passages)`. | called from COMPOSE task | returns `AnswerDraft` / `List<Citation>` |
| `CitationGuardrail` | `after-llm-response` guardrail (registered on `CitationAgent`) | Inspects the COMPOSE task's returned `Answer`. For every `Claim`, verifies that `claim.citedPassageId` is non-null and resolves to a passage in the upstream `PassageSet`. Rejects the answer on any citation violation; the agent loop retries within its 4-iteration budget. | COMPOSE task response | accept / structured-reject |
| `CoverageScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `Answer`, `ClaimSet`, `PassageSet`. Output: `CitationScore{score, rationale}`. | called from `evalStep` | returns score |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Passage(
    String passageId,
    String source,
    String documentTitle,
    String text,
    double relevanceScore,
    Instant retrievedAt
) {}

record PassageSet(List<Passage> passages, Instant retrievedAt) {}

record Claim(
    String claimId,
    String text,
    String citedPassageId,   // MUST resolve to a Passage.passageId in the upstream PassageSet
    double confidence
) {}

record ClaimSet(List<Claim> claims, Instant attributedAt) {}

record Citation(String claimId, String passageId, String passageSnippet) {}

record Answer(
    String queryId,
    String queryText,
    String responseBody,
    List<Claim> claims,
    List<Citation> citations,
    Instant composedAt
) {}

record CitationScore(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record QueryRecord(
    String queryId,
    Optional<String> queryText,
    Optional<PassageSet> passages,
    Optional<ClaimSet> claimSet,
    Optional<Answer> answer,
    Optional<CitationScore> citationScore,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum QueryStatus {
    CREATED, RETRIEVING, RETRIEVED, ATTRIBUTING, ATTRIBUTED,
    COMPOSING, COMPOSED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QueryCreated`, `RetrieveStarted`, `PassagesRetrieved`, `AttributeStarted`, `ClaimsAttributed`, `ComposeStarted`, `AnswerComposed`, `CitationScored`, `CitationGuardrailRejected`, `QueryFailed`.

Every nullable lifecycle field on the `QueryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ queryText }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query record.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Citation Query Engine</title>`.

The App UI tab is a two-column layout: a left rail with the live list of queries (status pill + query text + age) and a right pane with the selected query's detail — query text, retrieved passages table, attributed claims list, composed answer with inline citations, citation-coverage score chip, and a guardrail-rejection log strip if any citation-violation rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `after-llm-response` guardrail (citation-gate)**: `CitationGuardrail` is registered on `CitationAgent` and runs after the COMPOSE task returns an `Answer` but before that answer is committed to the entity. It iterates over `answer.claims` and checks that each `claim.citedPassageId` is non-null and is present in `PassageSet.passages[].passageId`. A single uncited claim or a dangling citation id causes the whole answer to be rejected. The guardrail returns a structured `uncited-claim` error naming every violating claim to the agent loop; the agent retries within its 4-iteration budget. On reject, the guardrail also calls `QueryEntity.recordCitationGuardrailRejection(claimId, reason)` so the rejection is visible in the UI's rejection-log strip and in the audit log.
- **E1 — `on-decision-eval`**: runs immediately after `AnswerComposed` lands, as `evalStep` inside the workflow. `CoverageScorer` is a deterministic rule-based scorer (no LLM call): (1) full attribution — every `Claim.citedPassageId` is non-null, (2) resolved citations — every cited `passageId` appears in `PassageSet`, (3) non-empty answer — `claims.size() > 0`, (4) no decoration citations — every cited passage contributes at least one claim. Emits `CitationScored{score:1..5, rationale}` on a one-point-per-rule basis with a sentence naming the largest gap. Scores ≤ 2 are flagged in the UI.

## 9. Agent prompts

- `CitationAgent` → `prompts/citation-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, call only the tools registered for that phase, treat each task's typed input as the entire context for that phase, and return the task's typed output with every claim properly attributed.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded query `What are the main findings on transformer attention efficiency?`; within 60 s the query reaches `EVALUATED` with non-empty passages, ≥ 2 claims, each claim citing a valid passage, and a citation-coverage score chip on the card.
2. **J2** — The agent's first iteration on a query returns an `Answer` with a claim whose `citedPassageId` is null (mock LLM path). `CitationGuardrail` rejects the answer; a `CitationGuardrailRejected` event lands on the entity; the agent retries and produces a fully-cited answer; the query eventually completes correctly. The UI's rejection-log strip shows the one rejected response.
3. **J3** — A query whose mock-LLM trajectory produces a claim citing a `passageId` absent from the recorded `PassageSet` is scored 1 with a rationale naming the dangling citation; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-query trace (logged at `INFO`); the RETRIEVE task's log shows only RETRIEVE-tool calls, the ATTRIBUTE task's log shows only ATTRIBUTE-tool calls, the COMPOSE task's log shows only COMPOSE-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named citation-rag demonstrating the sequential-pipeline x research-intel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-research-intel-citation-rag. Java package
io.akka.samples.citationqueryengineworkflow. Akka 3.6.0. HTTP port 9234.

Components to wire (exactly):

- 1 AutonomousAgent CitationAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/citation-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  RETRIEVE, ATTRIBUTE, and COMPOSE tool sets are ALL registered on the agent; citation gating is
  the job of CitationGuardrail, NOT of conditional .tools(...) wiring. The after-llm-response
  guardrail (CitationGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow CitationQueryWorkflow per queryId with four steps:
  * retrieveStep — emits RetrieveStarted on the entity, then calls componentClient
    .forAutonomousAgent(CitationAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions("Query: " + queryText + "\nPhase: RETRIEVE\nUse the search and fetch
      tools to retrieve 4-8 relevant passages about this query.")
        .metadata("queryId", queryId)
        .metadata("phase", "RETRIEVE")
        .taskType(CitationTasks.RETRIEVE_PASSAGES)
    ). Reads forTask(taskId).result(RETRIEVE_PASSAGES) to get PassageSet. Writes
    QueryEntity.recordPassages(passageSet). WorkflowSettings.stepTimeout 60s.
  * attributeStep — emits AttributeStarted, then runSingleTask with TaskDef.instructions
    (formatAttributeContext(passageSet, queryText)) and metadata.phase = "ATTRIBUTE", taskType
    ATTRIBUTE_CLAIMS. Writes QueryEntity.recordClaims(claimSet). stepTimeout 60s.
  * composeStep — emits ComposeStarted, then runSingleTask with TaskDef.instructions
    (formatComposeContext(claimSet, passageSet, queryText)) and metadata.phase = "COMPOSE", taskType
    COMPOSE_ANSWER. Writes QueryEntity.recordAnswer(answer). stepTimeout 60s.
  * evalStep — runs the deterministic CoverageScorer over (answer, claimSet, passageSet)
    and writes QueryEntity.recordCitationScore(citationScore). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(CitationQueryWorkflow::error). The error step writes QueryFailed
  and ends.

- 1 EventSourcedEntity QueryEntity (one per queryId). State QueryRecord{queryId,
  queryText: Optional<String>, passages: Optional<PassageSet>, claimSet: Optional<ClaimSet>,
  answer: Optional<Answer>, citationScore: Optional<CitationScore>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  QueryStatus enum: CREATED, RETRIEVING, RETRIEVED, ATTRIBUTING, ATTRIBUTED, COMPOSING, COMPOSED,
  EVALUATED, FAILED. Events: QueryCreated{queryText}, RetrieveStarted, PassagesRetrieved{passages},
  AttributeStarted, ClaimsAttributed{claimSet}, ComposeStarted, AnswerComposed{answer},
  CitationScored{citationScore}, CitationGuardrailRejected{claimId, reason}, QueryFailed{reason}.
  Commands: create, startRetrieve, recordPassages, startAttribute, recordClaims, startCompose,
  recordAnswer, recordCitationScore, recordCitationGuardrailRejection, fail, getQuery.
  emptyState() returns QueryRecord.initial("") with all Optional fields as Optional.empty() and
  guardrailRejections = List.of() — no commandContext() reference (Lesson 3).

- 1 View QueryView with row type QueryRow that mirrors QueryRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes QueryEntity events. ONE query
  getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {queryText}; mints queryId; calls
    QueryEntity.create(queryText); then starts CitationQueryWorkflow with id
    "citation-" + queryId; returns {queryId}), GET /queries (list from getAllQueries, sorted
    newest-first), GET /queries/{id} (one row), GET /queries/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- CitationTasks.java declaring three Task<R> constants:
    RETRIEVE_PASSAGES = Task.name("Retrieve passages").description("Search and fetch relevant
      passages for the query from the in-process corpus").resultConformsTo(PassageSet.class);
    ATTRIBUTE_CLAIMS = Task.name("Attribute claims").description("Extract factual claims from
      passages and link each claim to its source passage").resultConformsTo(ClaimSet.class);
    COMPOSE_ANSWER = Task.name("Compose answer").description("Draft a grounded answer whose
      every claim cites a retrieved passage").resultConformsTo(Answer.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {RETRIEVE, ATTRIBUTE, COMPOSE}. Each function-tool method carries a
  phase tag readable by CitationGuardrail (e.g. a custom annotation or a phase-registry map
  built at startup).

- RetrieveTools.java — @FunctionTool searchPassages(String query) -> List<Passage> reading
  from src/main/resources/sample-data/corpus/*.json keyed by topic slug; @FunctionTool
  fetchPassage(String passageId) -> Passage reading from the matching corpus entry.

- AttributeTools.java — @FunctionTool extractClaims(List<Passage>) -> List<Claim> (one Claim
  per Passage, claimId minted as "cl-" + sha1(text).substring(0,8), citedPassageId =
  passage.passageId); @FunctionTool linkClaims(List<Claim>, List<Passage>) -> List<Claim>
  (re-links each claim to its highest-scoring passage if ambiguous, preserving the original
  citedPassageId when unambiguous).

- ComposeTools.java — @FunctionTool draftAnswer(List<Claim>, String query) -> AnswerDraft
  (answer body composed from the claim texts with inline citation markers); @FunctionTool
  formatCitations(List<Claim>, List<Passage>) -> List<Citation> (one Citation per distinct
  citedPassageId, snippet from the passage text).

- CitationGuardrail.java — implements the after-llm-response hook. Fired when the COMPOSE
  task returns an Answer. Iterates answer.claims; for each claim, checks that
  claim.citedPassageId is non-null and that the id appears in passageSet.passages[].passageId
  (looked up by queryId from QueryEntity, carried in the TaskDef metadata). On any violation,
  returns Guardrail.reject("uncited-claim: claim <claimId> has no valid citation in the
  recorded PassageSet") and calls QueryEntity.recordCitationGuardrailRejection(claimId,
  reason). On full pass, returns accept.

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: Answer, ClaimSet,
  PassageSet. Outputs: CitationScore with score and rationale. Four checks, one point per
  check satisfied, starting from base 1: (1) full attribution — every claim.citedPassageId
  non-null, (2) resolved citations — every cited passageId in PassageSet, (3) non-empty
  answer — claims.size() > 0, (4) no decoration citations — every passage cited by at least
  one claim. Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9234 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/queries.jsonl with 5 seeded query lines covering the three
  surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/corpus/*.json — three files keyed by seeded query topic slug,
  each carrying 6-10 Passage entries with deterministic content so RetrieveTools.searchPassages
  returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — research-intel
  domain, baseline tier.

- risk-survey.yaml at the project root with data.data_classes.pii = false (queries are topic-
  level, not person-level), decisions.authority_level = recommend-only (the answer is advisory),
  oversight.human_in_loop = true (a researcher reads the answer before citing it),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "uncited-claim", "dangling-citation", "decoration-citation",
  "empty-claim-set"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/citation-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Citation Query Engine", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left = live list
  of query cards; right = selected-query detail with query text header, passages table, claims
  list with attribution, answer with inline citations, citation-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Citation Query Engine</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
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
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per call
  (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    retrieve-passages.json — 6 PassageSet entries, each with 5-8 Passage items per seeded
      query. Each entry's tool_calls array contains 2-4 calls: 1 searchPassages(query) + 1-3
      fetchPassage(passageId) calls.
    attribute-claims.json — 6 ClaimSet entries paired one-to-one with the retrieve entries,
      each with 3-6 Claim items, all citedPassageIds resolving to passages in the paired
      PassageSet. tool_calls containing extractClaims + linkClaims in order.
    compose-answer.json — 6 Answer entries paired one-to-one. Each carries 3-6 Claims with
      valid citedPassageIds and a matching Citations list. Plus 1 deliberately UNCITED-CLAIM
      entry whose first Claim has citedPassageId = null — CitationGuardrail rejects it; J2
      verifies this. Plus 1 DANGLING-CITATION entry whose first Claim cites a passageId absent
      from the paired PassageSet — CoverageScorer scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CitationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CitationTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (retrieveStep
  60s, attributeStep 60s, composeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the QueryRecord row record is Optional<T>.
- Lesson 7: CitationTasks.java with RETRIEVE_PASSAGES, ATTRIBUTE_CLAIMS, COMPOSE_ANSWER
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated model names.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9234 declared explicitly in application.conf's akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index.
  The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CitationAgent). The
  on-decision eval is rule-based (CoverageScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: all three phase tool sets are registered on the agent;
  CitationGuardrail is the post-response citation gate, not a phase-order gate.
- Task dependency is carried by typed task results: retrieveStep writes PassageSet onto the
  entity, attributeStep reads it and builds the ATTRIBUTE task's instruction context from it,
  composeStep reads both. The agent itself is stateless across phases.
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
