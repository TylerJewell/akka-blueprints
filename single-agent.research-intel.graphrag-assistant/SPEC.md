# SPEC — graphrag-assistant

The natural-language brief `/akka:specify @SPEC.md` reads to generate this
system. Sections 1–11 together are the input; Section 12 drives the rest of SDD.

---

## 1. System name + pitch

**System name:** GraphRAG Assistant.
**One-line pitch:** The user types a research question; one agent picks local or
global retrieval over an indexed corpus and returns a grounded answer with
citations.

## 2. What this blueprint demonstrates

The single-agent coordination pattern: one request/response Agent equipped with
two retrieval tools (local entity-neighborhood search and global
community-summary search) decides per query which scope fits, retrieves context,
and answers. The governance pattern is a before-agent-response grounding
guardrail that blocks answers not supported by retrieved context, plus a PII
sanitizer that scrubs personal data out of corpus chunks before they enter the
prompt.

## 3. User-facing flows

1. The user submits a question on the App UI tab (or `POST /api/ask`). A `QueryEntity` is created in `RECEIVED` state and the question appears in the live list over SSE.
2. `ResearchAgent` runs: it selects a retrieval scope, calls the matching tool against `CorpusIndex`, and drafts an answer with citations. The sanitizer scrubs PII from retrieved chunks before they reach the prompt.
3. The grounding guardrail checks the answer against the retrieved context. If grounded, the query moves to `ANSWERED` with answer text, scope, and citations. If not, it moves to `BLOCKED` with a reason.
4. The user reads the answer, the chosen scope (local/global), and the source citations in the live list.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ResearchAgent | Agent | Selects scope, calls a retrieval tool, drafts grounded answer | QueryEndpoint | CorpusIndex |
| CorpusIndex | KeyValueEntity | Holds in-memory graph index; serves local + global search | IndexBuilder, ResearchAgent | — |
| QueryEntity | EventSourcedEntity | Tracks one question's lifecycle | QueryEndpoint | QueriesView |
| QueriesView | View | Read model of all queries for UI + SSE | QueryEntity | QueryEndpoint |
| IndexBuilder | TimedAction | Builds the index from sample docs once at startup | — | CorpusIndex |
| QueryEndpoint | HttpEndpoint | Ask, list, single, SSE, metadata | UI / client | QueryEntity, QueriesView, ResearchAgent |
| AppEndpoint | HttpEndpoint | Serves the single-file UI | Browser | static-resources |

Names are used verbatim by `/akka:specify`.

## 5. Data model

Java records (full field list in `reference/data-model.md`). Every field null
before its transition is `Optional<T>` (Lesson 6).

- `Query(String id, String question, QueryStatus status, Instant createdAt, Optional<String> scope, Optional<Integer> chunkCount, Optional<Instant> retrievedAt, Optional<String> answer, Optional<Boolean> grounded, Optional<List<String>> citations, Optional<Instant> answeredAt, Optional<String> blockedReason)` — `QueryEntity` state and `QueriesView` row.
- `QueryStatus` enum: `RECEIVED`, `ANSWERED`, `BLOCKED`.
- `Answer(String text, String scope, boolean grounded, List<String> citations, int chunkCount)` — agent return type.
- `RetrievalResult(List<String> chunks, List<String> citations)` — `CorpusIndex` search return type.
- `IndexState(boolean built, int docCount, int entityCount, int communityCount)` — `CorpusIndex` state.

Events on `QueryEntity`: `QueryReceived(id, question, createdAt)`,
`AnswerRecorded(answer, scope, chunkCount, citations, answeredAt)`,
`AnswerBlocked(reason, blockedAt)`.

## 6. API contract

Full schemas in `reference/api-contract.md`. Top-level surface:

```
POST /api/ask                  -> { queryId }
GET  /api/queries              -> { queries: [Query, ...] }
GET  /api/queries/{id}         -> Query
GET  /api/queries/sse          -> Server-Sent Events of Query
GET  /api/metadata/eval-matrix -> text/yaml
GET  /api/metadata/risk-survey -> text/yaml
GET  /api/metadata/readme      -> text/markdown
GET  /                         -> 302 /app/index.html
GET  /app/{*path}              -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` — no
`ui/` folder, no npm build. Browser title `<title>Akka Sample: GraphRAG Assistant</title>`.
Five tabs: **Overview** (eyebrow + headline + Try it / How it works / Components /
API contract cards), **Architecture** (the four mermaid diagrams with the Lesson 24
CSS overrides and theme variables), **Risk Survey** (`risk-survey.yaml` in
`matrix-card`/`matrix-row` style; deployer placeholders muted), **Eval Matrix**
(`eval-matrix.yaml` with a colored mechanism pill per control), **App UI** (ask
box, live SSE query list, per-query scope/grounded/citations). Tab switching
matches by `data-tab`/`data-panel` attribute, never NodeList index (Lesson 26);
no hidden zombie panels. Detail in `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **G1 — grounding guardrail (before-agent-response, blocking).** Before
  `ResearchAgent`'s answer returns, verify every claim is supported by the
  retrieved context; block and record `BLOCKED` if grounding is insufficient.
- **S1 — PII sanitizer.** Scrub personal data from retrieved corpus chunks
  before they are injected into the agent prompt.

## 9. Agent prompts

- `ResearchAgent` → `prompts/research-agent.md` — selects retrieval scope, calls the matching tool, and answers only from retrieved context with citations.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. A specific entity question resolves via local search, returns `ANSWERED` with citations and `scope = "local"`.
2. A broad thematic question resolves via global search, returns `ANSWERED` with `scope = "global"`.
3. A question with no supporting corpus context returns `BLOCKED` with a grounding reason — no fabricated answer.
4. A corpus chunk containing PII never appears verbatim in the returned answer.

---

## 11. Implementation directives

```
Create a sample named graphrag-assistant demonstrating the single-agent ×
research-intel cell. Runs out of the box (no external services; in-memory graph
index built from bundled docs). Maven group io.akka.samples. Maven artifact
graphrag-assistant. Java package io.akka.samples.graphragassistant. Akka 3.6.0.
HTTP port 9986.

Components to wire (exactly):
- 1 Agent ResearchAgent. A request/response Agent (extends akka.javasdk.agent.Agent;
  NEVER an AutonomousAgent — single-agent request/response is the point). One
  method answer(String question) returning Effect<Answer> via
  effects().systemMessage(<prompts/research-agent.md>).userMessage(question)
  .responseAs(Answer.class).thenReply(). Register two function tools on the agent:
  localSearch(query) and globalSearch(query), each calling
  componentClient.forKeyValueEntity("corpus").method(CorpusIndex::localSearch /
  ::globalSearch).invoke(query) and returning RetrievalResult. The agent chooses
  which tool to call per the prompt's scope-selection rule.
- 1 KeyValueEntity CorpusIndex (entity id "corpus"). State IndexState{built,
  docCount, entityCount, communityCount}. Commands: build(List<String> docPaths)
  parses sample-docs into a simple entity/community graph and marks built=true;
  localSearch(String query) returns RetrievalResult by in-memory cosine/keyword
  match over entity-level chunks; globalSearch(String query) returns
  RetrievalResult over community-summary chunks; getStatus() returns IndexState.
  emptyState() returns IndexState(false,0,0,0) with no commandContext() reference.
- 1 EventSourcedEntity QueryEntity. State Query record with QueryStatus enum and
  Optional lifecycle fields (scope, chunkCount, retrievedAt, answer, grounded,
  citations, answeredAt, blockedReason). Events QueryReceived, AnswerRecorded,
  AnswerBlocked. Commands receive(question), recordAnswer(Answer), block(reason),
  getQuery. emptyState() returns Query.initial("","") with no commandContext().
- 1 View QueriesView with row type Query, TableUpdater consuming QueryEntity
  events. ONE query: getAllQueries SELECT * AS queries FROM queries_view. No
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side.
- 1 TimedAction IndexBuilder, runs once shortly after startup: lists
  src/main/resources/sample-docs/*.md, calls CorpusIndex.build, then self-disables
  (checks getStatus().built before rebuilding).
- 2 HttpEndpoints: QueryEndpoint at /api with ask (creates QueryEntity, calls
  ResearchAgent.answer, applies sanitizer S1 to retrieved chunks and guardrail G1
  to the answer, then recordAnswer or block), queries list (client-side filter
  over getAllQueries), single query, SSE stream (serverSentEventsForView), and
  three /api/metadata/* endpoints serving files from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- Records: Answer(String text, String scope, boolean grounded, List<String>
  citations, int chunkCount); RetrievalResult(List<String> chunks, List<String>
  citations); IndexState(...); Query(...).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9986
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-docs/ with 6 short markdown docs forming a small
  entity/community graph (one of them contains an obvious PII string to exercise
  S1). Include a doc index manifest if helpful.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (classpath copies of the root files for the metadata endpoints).
- eval-matrix.yaml at the project root with controls G1 and S1 plus a matching
  simplified_view. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, oversight; marking deployer-specific
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per Section 12 of the authoring guide. No
  governance-mechanisms section, no configuration section.
- src/main/resources/static-resources/index.html — one self-contained file (no
  ui/, no npm). Inline CSS + JS; runtime CDN imports for markdown, YAML, and
  mermaid acceptable. Five tabs per Section 7. Match the governance.html visual
  style (dark / yellow accent / Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider returning shape-correct outputs per
  agent (ResearchAgent -> Answer); set model-provider = mock; write
  src/main/resources/mock-responses/research-agent.json with 4-6 entries
  spanning local-scope, global-scope, and a blocked/ungrounded case.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources it before spawning the JVM.
  (d) Secrets-store URI — record the URI in .akka-build.yaml; resolve at run time.
  (e) Type once in this session — value lives only in the Claude session; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the REFERENCE.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echo any captured key.

Mock LLM provider — per-agent response shapes:
- ResearchAgent -> Answer{text, scope in ["local","global"], grounded boolean,
  citations (1-3 sample-doc ids), chunkCount 1-4}. Include at least one entry with
  grounded=false and empty citations to exercise the G1 block path.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: ResearchAgent extends Agent (request/response) — never downgraded or
  upgraded to a different primitive.
- Lesson 4: any step calling the agent uses an explicit timeout >= 60s (the
  endpoint's component-client call inherits this; no 5s default).
- Lesson 6: Optional<T> for every nullable lifecycle field on the Query row.
- Lesson 7: no AutonomousAgent here, so no Tasks.java — do not generate one.
- Lesson 8: verify model names current before locking them in application.conf.
- Lesson 9: run command is "/akka:build" (Claude Code slash command), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9986 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: descriptive integration label "Runs out of the box" — never T1-T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: static-resources/index.html includes the mermaid state-label CSS
  overrides + edge-label foreignObject overflow:visible + transitionLabelColor
  #cccccc.
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, never NodeList
  index; delete removed panels, never display:none zombies.
- emptyState() never calls commandContext() (Lesson 3).
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into
`specs/features/`, **do not stop and wait for the user**. Immediately continue
through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening
URL and a one-line summary of any failures from step 3. Do not narrate
intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around
without asking the user — a missing API key with no provider env var set and no
mock chosen, an unrecoverable compile error after exhausting auto-fix options, or
a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
