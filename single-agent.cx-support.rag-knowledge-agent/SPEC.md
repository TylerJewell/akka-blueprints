# SPEC — rag-knowledge-agent

The natural-language brief `/akka:specify @SPEC.md` reads to generate this system. Sections 1–11 together are the input; Section 12 drives the rest of the pipeline.

---

## 1. System name + pitch

**System name:** RAG Knowledge Agent.
**One-line pitch:** The user types a question into the App UI; the agent retrieves passages from a fixed document set, answers using only those passages with inline citations, and refuses when the documents do not support an answer.

## 2. What this blueprint demonstrates

The single-agent retrieval-augmented question-answering pattern: one request/response agent grounded on a retrieved passage set, with no multi-agent orchestration. The governance pattern is grounded refusal plus per-answer faithfulness scoring — a before-agent-response guardrail blocks answers not supported by the retrieved passages, and an on-decision eval scores every produced answer against the passages it claims to cite.

## 3. User-facing flows

1. The user opens the App UI tab and types a question. The system records a query session in `RECEIVED` and retrieves the top passages from the document set; the session moves to `RETRIEVED`.
2. The agent answers using the retrieved passages. If the answer is supported, the session becomes `ANSWERED` and shows the answer with citations. If not, the grounding guardrail forces a refusal and the session becomes `REFUSED`.
3. Once answered, a faithfulness eval scores the answer against the retrieved passages; the session becomes `EVALUATED` and the score appears next to the answer.
4. The document set loads at startup from the bundled files; the user can ask questions immediately.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| KnowledgeEndpoint | HttpEndpoint | Accept questions, orchestrate retrieve→answer, serve list/SSE/metadata | browser | DocIndexEntity, KnowledgeAgent, QuerySessionEntity, QuerySessionView |
| AppEndpoint | HttpEndpoint | Serve the static UI | browser | static-resources |
| DocIndexEntity | KeyValueEntity | In-memory chunk store; `search(embedding, topK)` cosine command | DocLoader, KnowledgeEndpoint | — |
| DocLoader | TimedAction | Load + embed `sample-docs/*` into DocIndexEntity at startup | scheduler | DocIndexEntity |
| KnowledgeAgent | Agent | Grounded QA; returns answer + citations or refusal | KnowledgeEndpoint | QuerySessionEntity |
| FaithfulnessAgent | Agent | Score answer faithfulness against retrieved passages | FaithfulnessEvalConsumer | QuerySessionEntity |
| QuerySessionEntity | EventSourcedEntity | Per-question lifecycle and events | KnowledgeEndpoint, FaithfulnessEvalConsumer | QuerySessionView |
| QuerySessionView | View | Read model for list + SSE | QuerySessionEntity events | KnowledgeEndpoint |
| FaithfulnessEvalConsumer | Consumer | On `QueryAnswered`, run FaithfulnessAgent, record score | QuerySessionEntity events | FaithfulnessAgent, QuerySessionEntity |

Names are used verbatim by `/akka:specify`.

## 5. Data model

See `reference/data-model.md`. Authoritative records:

- `QuerySession(String id, String question, QueryStatus status, Instant receivedAt, List<Citation> retrievedChunks, Optional<Instant> retrievedAt, Optional<String> answer, List<Citation> citations, Optional<Boolean> grounded, Optional<Instant> answeredAt, Optional<String> refusalReason, Optional<Double> faithfulnessScore, Optional<String> faithfulnessVerdict, Optional<Instant> evaluatedAt)`. Every nullable lifecycle field is `Optional<T>` (Lesson 6); list fields default to empty.
- `Citation(String docId, String docTitle, String chunkId, String snippet, double score)`.
- `Chunk(String docId, String docTitle, String chunkId, String text, List<Double> embedding)`.
- `QueryStatus` enum: `RECEIVED, RETRIEVED, ANSWERED, REFUSED, EVALUATED`.
- Events: `QueryReceived, ChunksRetrieved, QueryAnswered, QueryRefused, FaithfulnessEvaluated`.

## 6. API contract

See `reference/api-contract.md` for schemas. Top-level surface:

```
POST /api/ask                      -> { sessionId }
GET  /api/sessions ?status=...     -> { sessions: [QuerySession, ...] }
GET  /api/sessions/{id}            -> QuerySession
GET  /api/sessions/sse             -> Server-Sent Events of QuerySession
GET  /api/metadata/eval-matrix     -> text/yaml
GET  /api/metadata/risk-survey     -> text/yaml
GET  /api/metadata/readme          -> text/markdown
GET  /                             -> 302 /app/index.html
GET  /app/{*path}                  -> static-resources/{*path}
```

## 7. UI

Five tabs — Overview / Architecture / Risk Survey / Eval Matrix / App UI — in one self-contained `src/main/resources/static-resources/index.html`. Browser title: `<title>Akka Sample: RAG Knowledge Agent</title>`. Overview is eyebrow ("Overview") + headline (sample type) with no subtitle, then the four cards Try it / How it works / Components / API contract. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, with no hidden zombie panels (Lesson 26). The Architecture tab renders the mermaid diagrams with the state-label CSS overrides and theme variables from Lesson 24. App UI: a question box + Ask button, a live session list via SSE, each answered session showing the answer, its citations, and its faithfulness score; refused sessions show the refusal reason. See `reference/ui-mockup.md`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **G1 — grounding guardrail (guardrail · before-agent-response).** Before `KnowledgeAgent`'s answer is returned, a grounding check verifies each claim is supported by the retrieved passages; unsupported answers are converted to a refusal (`QueryRefused`).
- **E1 — faithfulness eval (eval-event · on-decision-eval).** On `QueryAnswered`, `FaithfulnessEvalConsumer` runs `FaithfulnessAgent` to score the answer against its retrieved passages and records `FaithfulnessEvaluated`.

## 9. Agent prompts

- `prompts/knowledge-agent.md` — grounded QA: answer only from supplied passages, cite, refuse when unsupported.
- `prompts/faithfulness-agent.md` — score how faithful a given answer is to its retrieved passages.

## 10. Acceptance

See `reference/user-journeys.md`. Passing means:

1. A question answerable from the document set returns an `ANSWERED` session with a non-empty answer and at least one citation.
2. A question with no support in the document set returns a `REFUSED` session with a refusal reason.
3. Every answered session reaches `EVALUATED` with a faithfulness score between 0 and 1.
4. On startup, `DocLoader` populates `DocIndexEntity` so the first question retrieves passages.

---

## 11. Implementation directives

The whole SPEC.md (Sections 1–11) is the input to `/akka:specify @SPEC.md`.

```
Create a sample named rag-knowledge-agent demonstrating the single-agent ×
cx-support cell. Runs out of the box (no external services; in-process
retrieval). Maven group io.akka.samples. Maven artifact rag-knowledge-agent.
Java package io.akka.samples.ragknowledgeagent. Akka 3.6.0. HTTP port 9443.

Components to wire (exactly):
- 1 Agent KnowledgeAgent: method answer(AnswerRequest{question, chunks}) using
  effects().systemMessage(prompts/knowledge-agent.md).userMessage(...)
  .responseAs(GroundedAnswer.class).thenReply(). GroundedAnswer{String answer,
  List<Citation> citations, boolean grounded, String refusalReason}. The agent
  must only answer from supplied chunks and set grounded=false with a
  refusalReason when the chunks do not support an answer.
- 1 Agent FaithfulnessAgent: method score(ScoreRequest{answer, chunks})
  returning FaithfulnessResult{double score, String verdict}. score in [0,1].
- 1 KeyValueEntity DocIndexEntity holding DocIndex{List<Chunk> chunks}.
  Commands: indexChunks(List<Chunk>), search(SearchQuery{List<Double>
  queryEmbedding, int topK}) -> List<Citation> by cosine similarity computed
  in-process; getChunks(). Single well-known entity id "default".
- 1 TimedAction DocLoader (every 60s, idempotent): reads
  src/main/resources/sample-docs/*.md, splits into ~500-char chunks, computes a
  deterministic lexical embedding per chunk, and calls DocIndexEntity.indexChunks.
  No-op when the index already covers the current files.
- 1 EventSourcedEntity QuerySessionEntity holding QuerySession (id, question,
  QueryStatus enum, receivedAt, retrievedChunks list, plus Optional lifecycle
  fields retrievedAt, answer, grounded, answeredAt, refusalReason,
  faithfulnessScore, faithfulnessVerdict, evaluatedAt and a citations list).
  Events: QueryReceived, ChunksRetrieved, QueryAnswered, QueryRefused,
  FaithfulnessEvaluated. Commands: receive(question), recordRetrieval(chunks),
  recordAnswer(answer, citations), recordRefusal(reason), recordEvaluation(
  score, verdict), getSession. emptyState() returns QuerySession.initial("")
  with no commandContext() reference.
- 1 View QuerySessionView with row type QuerySession, table updater consuming
  QuerySessionEntity events. ONE query: getAllSessions SELECT * AS sessions
  FROM query_sessions. No WHERE status filter (Akka cannot auto-index enum
  columns) — filter client-side in callers.
- 1 Consumer FaithfulnessEvalConsumer subscribed to QuerySessionEntity events;
  on QueryAnswered, calls FaithfulnessAgent.score with the answer and the
  session's retrievedChunks, then calls QuerySessionEntity.recordEvaluation.
- 2 HttpEndpoints: KnowledgeEndpoint at /api with ask (creates session, embeds
  the question via the same lexical embedding, calls DocIndexEntity.search,
  records retrieval, calls KnowledgeAgent.answer, records answer or refusal),
  sessions list (filter client-side from getAllSessions), single session, SSE
  stream, and three /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.

Companion files:
- Records: AnswerRequest, GroundedAnswer, ScoreRequest, FaithfulnessResult,
  Citation, Chunk, DocIndex, SearchQuery, QuerySession as above.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9443 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-docs/ with 3-4 short markdown docs forming the
  knowledge set (device setup, account, troubleshooting), plus enough off-topic
  gaps that some questions are unanswerable.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files served from the classpath).
- eval-matrix.yaml at the project root with controls G1, E1 and a matching
  simplified_view. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, oversight; marking deployer-specific
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root per the authoring guide. NO governance-
  mechanisms section. NO configuration section.
- src/main/resources/static-resources/index.html — one self-contained file
  (no ui/ folder, no npm build). Inline CSS + JS; runtime CDN imports for
  markdown and YAML are acceptable. Five tabs: Overview, Architecture (mermaid
  with Lesson 24 overrides), Risk Survey (matrix-card/matrix-row; mute
  TO_BE_COMPLETED_BY_DEPLOYER), Eval Matrix (matrix-card/matrix-row; colored
  mechanism pill in the label column), App UI (ask box, SSE session list,
  answer + citations + faithfulness score, refusal reason). Match the
  governance.html visual style (dark/yellow/Instrument Sans/dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (KnowledgeAgent -> GroundedAnswer, FaithfulnessAgent ->
  FaithfulnessResult; write src/main/resources/mock-responses/
  {knowledge-agent,faithfulness-agent}.json with 4-6 entries each). Sets
  model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory only;
  passed to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the REFERENCE
  — env-var name, file path, or secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echoes any captured key.

Mock LLM provider per-agent shapes:
- KnowledgeAgent -> GroundedAnswer: pick a 1-2 sentence answer that quotes a
  supplied chunk and one Citation referencing that chunk, grounded=true; for the
  "unanswerable" mock entry set grounded=false and a refusalReason.
- FaithfulnessAgent -> FaithfulnessResult: a random score in [0.7,1.0] with
  verdict "supported", and one low-score entry (~0.3, "unsupported").

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: Agent stays Agent — never silently swapped for AutonomousAgent or
  vice versa.
- Lesson 4: every workflow/agent-calling step that calls an LLM gets an explicit
  timeout (60s) — applies to the endpoint's agent calls and the consumer.
- Lesson 6: Optional<T> for every nullable lifecycle field on the View row record.
- Lesson 7: AutonomousAgent companion Tasks.java only if an AutonomousAgent is
  used — this sample uses request/response Agents, so no Tasks.java.
- Lesson 8: verify model names current before locking in application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode http-port 9443.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column, no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label CSS overrides + theme variables in index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and the PLAN.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9443`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
