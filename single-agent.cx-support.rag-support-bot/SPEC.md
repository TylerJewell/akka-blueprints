# SPEC — rag-support-bot

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SupportFlow Lite.
**One-line pitch:** A customer submits a support query; one AI agent retrieves relevant knowledge-base passages and composes a grounded reply — every candidate reply is checked for passage anchoring before it reaches the customer.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the CX-support domain. One `SupportAgent` (AutonomousAgent) carries the entire reply-generation decision; the surrounding components prepare its input and validate its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw customer query and the agent call — so the model never sees the customer's name, email, order number, or other identifiers.
- A **before-agent-response guardrail** validates the agent's reply on every turn: well-formed `SupportReply`, at least one `sourcePassageId` cited in the reply, and no forbidden escalation language used without an `escalation` flag set. A reply that fails any check triggers a retry inside the same task.

Additionally, a lightweight **on-decision evaluator** runs after each `ReplyRecorded` event, scoring the reply for grounding quality (do the cited passage ids exist in the knowledge base? is the reply length within policy? does the reply avoid fabricated product names?). This evaluator is deterministic and rule-based — it does not make an LLM call, preserving the single-agent invariant.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a query into the **Ask a question** textarea (or picks one of four seeded example queries — about order status, returns policy, account password reset, and product compatibility).
2. The user optionally selects a **topic filter** from a dropdown (Orders, Returns, Account, Technical) to bias retrieval, or leaves it on `Any`.
3. The user clicks **Send**. The UI POSTs to `/api/conversations` and receives a `conversationId`.
4. The conversation card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `SANITIZED` — the sanitized query is visible in the card detail.
5. Within ~2 s, retrieval completes and the card transitions to `RETRIEVING` then `REPLYING`. Within ~10–30 s the card transitions to `REPLY_RECORDED`. The reply appears with: the answer text, a numbered list of source passages that anchor it, and a grounding score chip.
6. Within ~1 s the `evalStep` finishes. The card shows a **grounding score** chip (1–5) plus a one-line rationale.
7. The user can submit another query; the live list shows history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationEndpoint` | `HttpEndpoint` | `/api/conversations/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle: received → sanitized → retrieving → replying → reply-recorded → evaluated. Source of truth. | `ConversationEndpoint`, `MessageSanitizer`, `ConversationWorkflow` | `ConversationView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `QueryReceived` events; strips PII; calls `ConversationEntity.attachSanitized`. After sanitization, starts `ConversationWorkflow`. | `ConversationEntity` events | `ConversationEntity`, `ConversationWorkflow` |
| `KnowledgeBase` | `EventSourcedEntity` | One entity per article id. Stores article title, body, topic tags, and passage segments. Loaded from seeded JSONL at startup. | `AppEndpoint` seed-load | `ConversationWorkflow` (in-process retrieval) |
| `ConversationWorkflow` | `Workflow` | One workflow per conversation. Steps: `awaitSanitizedStep` → `retrieveStep` → `replyStep` → `evalStep`. | started by `MessageSanitizer` | `SupportAgent`, `ConversationEntity` |
| `SupportAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized query as task instructions and the retrieved passages as a task attachment; returns `SupportReply`. | invoked by `ConversationWorkflow` | returns reply |
| `ReplyGuardrail` | guardrail implementation | Registered on `SupportAgent` via before-agent-response hook. Validates passage citations, escalation flag consistency, and reply length. | — | rejects or passes `SupportReply` |
| `GroundingScorer` | scoring helper | Deterministic rule-based scorer. Runs inside `evalStep`. No LLM call. | — | emits `EvaluationScored` |
| `ConversationView` | `View` | Read model: one row per conversation for the UI. | `ConversationEntity` events | `ConversationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record QueryRequest(
    String conversationId,
    String rawQuery,
    String topicFilter,        // "ANY" | "ORDERS" | "RETURNS" | "ACCOUNT" | "TECHNICAL"
    String submittedBy,
    Instant receivedAt
) {}

record SanitizedQuery(
    String redactedQuery,
    List<String> piiCategoriesFound
) {}

record KbPassage(
    String passageId,
    String articleId,
    String articleTitle,
    String passageText,
    String topic
) {}

record RetrievalResult(
    List<KbPassage> passages,
    int passageCount
) {}

record SupportReply(
    String answerText,
    List<String> sourcePassageIds,
    boolean escalation,
    String escalationReason,     // empty if escalation==false
    ReplyOutcome outcome,
    Instant repliedAt
) {}
enum ReplyOutcome { ANSWERED, PARTIALLY_ANSWERED, ESCALATE }

record GroundingEval(
    int score,              // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Conversation(
    String conversationId,
    Optional<QueryRequest> query,
    Optional<SanitizedQuery> sanitized,
    Optional<RetrievalResult> retrieved,
    Optional<SupportReply> reply,
    Optional<GroundingEval> eval,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ConversationStatus {
    RECEIVED, SANITIZED, RETRIEVING, REPLYING, REPLY_RECORDED, EVALUATED, FAILED
}
```

Events on `ConversationEntity`: `QueryReceived`, `QuerySanitized`, `RetrievalCompleted`, `ReplyStarted`, `ReplyRecorded`, `EvaluationScored`, `ConversationFailed`.

Every nullable lifecycle field on the `Conversation` record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ rawQuery, topicFilter, submittedBy }` → `{ conversationId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation.
- `GET /api/conversations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SupportFlow Lite</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted conversations (status pill + outcome badge + age) and a right pane with the selected conversation's detail — sanitized query, retrieved passages, reply text with source citations, and the grounding-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): strips emails, phone numbers, order-number-like tokens, account-id-like identifiers, person names, and postal addresses from the raw customer query before any LLM call. Records which categories were found.
- **G1 — before-agent-response guardrail**: runs on every turn of `SupportAgent`. Asserts the candidate response is well-formed `SupportReply` JSON, at least one `sourcePassageId` in the reply is present in `retrieved.passages`, the `escalation` flag is set if and only if `outcome == ESCALATE`, and `answerText` is within the configured length limit. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `SupportAgent` → `prompts/support-agent.md`. The single decision-making LLM. System prompt instructs it to read the retrieved passages as its only source of truth and compose a reply that explicitly cites which passage ids ground each claim.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the order-status seed query; within 30 s the reply appears with at least one cited passage id and a grounding eval score.
2. **J2** — The agent's first response on a query cites a passage id that is not in the retrieved set (mock LLM path) — the guardrail rejects it; the second iteration produces a valid reply; the UI never displays the rejected response.
3. **J3** — A reply with no cited passages and a short `answerText` receives a grounding score of 1 with a clear rationale; the card border highlights red.
4. **J4** — A query containing `john.doe@example.com` and `order #ORD-98765` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-ORDER-ID]`; the entity's `query.rawQuery` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named rag-support-bot demonstrating the single-agent × cx-support cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-rag-support-bot. Java package
io.akka.samples.supportflowlitesimpleaicustomersupportchatbot. Akka 3.6.0. HTTP port 9423.

Components to wire (exactly):

- 1 AutonomousAgent SupportAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/support-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUERY).maxIterationsPerTask(3)). The task receives
  the sanitized query as its instruction text and the retrieved passages as a task ATTACHMENT
  named "passages.txt" (NOT as inline prompt text — Akka's TaskDef.attachment(name,
  contentBytes) is the canonical call). Output: SupportReply{answerText: String,
  sourcePassageIds: List<String>, escalation: boolean, escalationReason: String,
  outcome: ReplyOutcome, repliedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow ConversationWorkflow per conversationId with four steps:
  * awaitSanitizedStep — polls ConversationEntity.getConversation every 1s; on
    conversation.sanitized().isPresent() advances to retrieveStep.
    WorkflowSettings.stepTimeout 15s.
  * retrieveStep — performs in-process BM25-style keyword retrieval against KnowledgeBase
    entities (top-5 passages matching sanitized query tokens and topic filter); emits
    RetrievalCompleted{passages, passageCount} on entity; advances to replyStep.
    WorkflowSettings.stepTimeout 10s.
  * replyStep — emits ReplyStarted, then calls componentClient.forAutonomousAgent(
    SupportAgent.class, "support-" + conversationId).runSingleTask(
      TaskDef.instructions(formatQuery(conversation.sanitized.redactedQuery))
        .attachment("passages.txt", formatPassages(retrieved.passages).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANSWER_QUERY) to fetch the reply.
    On success calls ConversationEntity.recordReply(reply). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(ConversationWorkflow::error).
  * evalStep — runs a deterministic rule-based GroundingScorer (NOT an LLM call) over the
    recorded reply: checks that every sourcePassageId is in retrieved.passages, that
    answerText length is within policy (50–600 chars), that no passage is cited more than
    twice, and that escalation flag matches outcome. Emits EvaluationScored{score: 1-5,
    rationale: String}. WorkflowSettings.stepTimeout 5s. error step transitions the entity
    to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId: String, query: Optional<QueryRequest>,
  sanitized: Optional<SanitizedQuery>, retrieved: Optional<RetrievalResult>,
  reply: Optional<SupportReply>, eval: Optional<GroundingEval>,
  status: ConversationStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  ConversationStatus enum: RECEIVED, SANITIZED, RETRIEVING, REPLYING, REPLY_RECORDED,
  EVALUATED, FAILED. Events: QueryReceived{query}, QuerySanitized{sanitized},
  RetrievalCompleted{retrieved}, ReplyStarted{}, ReplyRecorded{reply},
  EvaluationScored{eval}, ConversationFailed{reason}. Commands: receive, attachSanitized,
  recordRetrieval, markReplying, recordReply, recordEvaluation, fail, getConversation.
  emptyState() returns Conversation.initial("") with no commandContext() reference (Lesson
  3). Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 EventSourcedEntity KnowledgeBase (one per articleId). State
  KbArticle{articleId: String, title: String, body: String, topic: String,
  passages: List<KbPassage>}. Command: load(KbArticle). Event: ArticleLoaded{article}.
  Passages are computed deterministically in the event-applier by splitting the body
  into 200-word segments, assigning passageId = articleId + "-p" + index.

- 1 Consumer MessageSanitizer subscribed to ConversationEntity events; on QueryReceived
  runs a regex+heuristic redaction pipeline (emails, phone numbers, order-number tokens
  ORD-\d+, account-id-like tokens, postal addresses, person-name heuristic) over rawQuery,
  computes the list of categories found, builds SanitizedQuery, then calls
  ConversationEntity.attachSanitized(sanitized). After attachSanitized lands, the same
  Consumer starts a ConversationWorkflow with id = "conv-" + conversationId.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation minus
  query.rawQuery — the audit log keeps the raw; the view holds the sanitized form).
  Table updater consumes ConversationEntity events. ONE query getAllConversations:
  SELECT * AS conversations FROM conversation_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with POST /conversations (body
    {rawQuery, topicFilter, submittedBy}; mints conversationId; calls
    ConversationEntity.receive; returns {conversationId}), GET /conversations (list from
    getAllConversations, sorted newest-first), GET /conversations/{id} (one row), GET
    /conversations/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SupportTasks.java declaring one Task<R> constant: ANSWER_QUERY = Task.name("Answer
  customer query").description("Read the retrieved passages and compose a SupportReply
  anchored in those passages").resultConformsTo(SupportReply.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records QueryRequest, SanitizedQuery, KbPassage, RetrievalResult, SupportReply,
  ReplyOutcome, GroundingEval, Conversation, ConversationStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  SupportReply from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>)
  to force the agent loop to retry.

- GroundingScorer.java — pure deterministic logic (no LLM). Inputs: SupportReply and the
  RetrievalResult used in that conversation. Outputs: GroundingEval. Scoring rubric
  documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9423 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The SupportAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/kb-articles.jsonl with 4 seeded FAQ articles:
  an Order Status FAQ (400 words, topic ORDERS), a Returns Policy FAQ (350 words, topic
  RETURNS), an Account Management FAQ (300 words, topic ACCOUNT), and a Technical
  Compatibility FAQ (450 words, topic TECHNICAL). Each article contains 2–3 plausible
  customer identifiers so S1 has work to do when they appear in example queries.

- src/main/resources/sample-events/seed-queries.jsonl with 4 seed customer queries:
  a query about order tracking (contains "order #ORD-12345" and an email), a query about
  return windows, a query about password reset, and a technical compatibility question.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = automated
  (replies are sent directly to the customer, not reviewed by a human first),
  oversight.human_in_loop = false, oversight.human_on_loop = true (a support manager
  can review logs), failure.failure_modes including "hallucinated-product-detail",
  "uncited-claim", "pii-leakage-via-llm", "incorrect-escalation-decision",
  "grounding-score-1-not-flagged-to-human"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/support-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: SupportFlow Lite", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of conversation cards; right = selected conversation detail with sanitized
  query, retrieved passages, reply text, source citation list, grounding-score chip).
  Browser title exactly: <title>Akka Sample: SupportFlow Lite</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversationId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-customer-query.json — 6 SupportReply entries covering the three ReplyOutcome
      values (ANSWERED, PARTIALLY_ANSWERED, ESCALATE). Each entry has a realistic
      answerText, a sourcePassageIds array referencing actual seeded passage ids, and
      appropriate escalation fields. Plus 2 deliberately MALFORMED entries (one with a
      sourcePassageId that does not exist in the retrieved set; one with outcome=ESCALATE
      but escalation=false) — the guardrail blocks both, exercising the retry path. The
      mock selects a malformed entry on the FIRST iteration of every 3rd conversation
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(conversationId) helper makes per-conversation selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SupportAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SupportTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (replyStep
  60s, awaitSanitizedStep 15s, retrieveStep 10s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Conversation row record is Optional<T>.
- Lesson 7: SupportTasks.java with ANSWER_QUERY = Task.name(...).description(...)
  .resultConformsTo(SupportReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9423 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (SupportAgent). The
  on-decision evaluator is rule-based (GroundingScorer.java) and does NOT make an LLM
  call.
- The retrieved passages are passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated replyStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
