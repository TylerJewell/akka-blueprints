# SPEC — ig-dm-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** IG DM Agent.
**One-line pitch:** An inbound Instagram direct message arrives; one AI agent reads the sanitized message plus the brand voice profile and returns a typed `DmReply` — a brand-safe, on-tone response ready to be dispatched back to the sender.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `DmReplyAgent` (AutonomousAgent) carries the entire response decision; the surrounding components only prepare its input and validate its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw inbound message and the agent call — so the model never sees phone numbers, email addresses, or personal identifiers that a customer may have included in their DM.
- A **before-agent-response guardrail** validates the agent's draft reply on every turn: the reply must not contain prohibited words or competitor brand mentions, must stay within the allowed character count, and must not make factual commitments (e.g., price guarantees, delivery dates) that are outside the approved response scope. A failing draft triggers a retry inside the same task.

The blueprint shows that the single-agent pattern does not mean "unmoderated" — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes an inbound DM into the **Message** textarea (or picks one of three seeded examples — a product availability inquiry, an order-status complaint, and a general brand question).
2. The user picks a **brand profile** from a dropdown (Retail, Hospitality, SaaS) or pastes a custom brand voice description.
3. The user clicks **Send to agent**. The UI POSTs to `/api/messages` and receives a `messageId`.
4. The card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted message is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `replyStep` completes. The card transitions to `REPLYING` then `REPLIED`. The draft reply appears: the response text, a tone-confidence score, and the brand-compliance pass/fail badge.
6. The user can submit another message; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DmEndpoint` | `HttpEndpoint` | `/api/messages/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `DmEntity`, `DmView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DmEntity` | `EventSourcedEntity` | Per-message lifecycle: received → sanitized → replying → replied. Source of truth. | `DmEndpoint`, `MessageSanitizer`, `DmWorkflow` | `DmView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; strips PII; calls `DmEntity.attachSanitized`. | `DmEntity` events | `DmEntity` |
| `DmWorkflow` | `Workflow` | One workflow per message. Steps: `awaitSanitizedStep` → `replyStep`. | started by `MessageSanitizer` once sanitized event lands | `DmReplyAgent`, `DmEntity` |
| `DmReplyAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the brand voice profile in the task definition and the sanitized message as a task attachment; returns `DmReply`. | invoked by `DmWorkflow` | returns reply |
| `ReplyGuardrail` | supporting class | Validates each candidate `DmReply` — no prohibited terms, no competitor mentions, within character limit, no out-of-scope commitments. | invoked by `DmReplyAgent` before-agent-response hook | accept or reject |
| `DmView` | `View` | Read model: one row per message for the UI. | `DmEntity` events | `DmEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BrandProfile(
    String profileId,
    String name,
    String voiceDescription,
    List<String> prohibitedTerms,
    List<String> competitorNames,
    int maxReplyChars,
    List<String> outOfScopeTopics
) {}

record InboundMessage(
    String messageId,
    String senderId,
    String rawText,
    String profileId,
    Instant receivedAt
) {}

record SanitizedMessage(
    String sanitizedText,
    List<String> piiCategoriesFound
) {}

record DmReply(
    String replyText,
    ToneLabel toneLabel,
    int toneConfidenceScore,  // 1..5
    String decidedAt           // ISO-8601 Instant
) {}
enum ToneLabel { FRIENDLY, PROFESSIONAL, EMPATHETIC, NEUTRAL }

record DmMessage(
    String messageId,
    Optional<InboundMessage> inbound,
    Optional<SanitizedMessage> sanitized,
    Optional<DmReply> reply,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED, SANITIZED, REPLYING, REPLIED, FAILED
}
```

Events on `DmEntity`: `MessageReceived`, `MessageSanitized`, `ReplyStarted`, `ReplyRecorded`, `MessageFailed`.

Every nullable lifecycle field on the `DmMessage` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/messages` — body `{ senderId, rawText, profileId }` → `{ messageId }`.
- `GET /api/messages` — list all messages, newest-first.
- `GET /api/messages/{id}` — one message.
- `GET /api/messages/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: IG DM Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of inbound messages (status pill + tone label + age) and a right pane with the selected message's detail — raw message (with PII categories listed), sanitized preview, draft reply, and tone-confidence score.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `DmReplyAgent`. Asserts the candidate `DmReply.replyText` contains no prohibited terms from the brand profile's `prohibitedTerms` list, no competitor brand names from `competitorNames`, stays within `maxReplyChars`, and does not include phrases matching the out-of-scope-commitment pattern (price-guarantee, delivery-date, refund-promise detected by a keyword ruleset). On failure, returns a structured rejection error to the agent loop so the task retries within its iteration budget.
- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): strips phone numbers, email addresses, URLs, government-identifier patterns, and full-name heuristics from the raw inbound DM before any LLM call. Records which categories were found.

## 9. Agent prompts

- `DmReplyAgent` → `prompts/dm-reply-agent.md`. The single decision-making LLM. System prompt instructs it to read the brand profile, read the sanitized inbound DM (passed as a task attachment), and return one `DmReply` with on-brand, on-tone response text.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded product-availability DM with the Retail brand profile; within 30 s the reply appears with a FRIENDLY or PROFESSIONAL tone label and a toneConfidenceScore ≥ 3.
2. **J2** — The agent's first draft (mock LLM path) contains a prohibited term; the `before-agent-response` guardrail rejects it; the second iteration produces a compliant reply; the UI never displays the rejected draft.
3. **J3** — A DM containing `user@example.com` and `+1-555-867-5309` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `inbound.rawText` retains the raw text for audit.
4. **J4** — A draft reply exceeding `maxReplyChars` is caught by the guardrail and a shorter compliant reply lands.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ig-dm-agent demonstrating the single-agent × cx-support cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-ig-dm-agent. Java package
io.akka.samples.aiagentforinstagramdminbox. Akka 3.6.0. HTTP port 9721.

Components to wire (exactly):

- 1 AutonomousAgent DmReplyAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/dm-reply-agent.md>) and
  .capability(TaskAcceptance.of(DmTasks.REPLY_TO_MESSAGE).maxIterationsPerTask(3)). The task
  receives the brand voice profile as its instruction text and the sanitized inbound DM as a
  task ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes)
  is the canonical call). Output: DmReply{replyText: String, toneLabel: ToneLabel (FRIENDLY/
  PROFESSIONAL/EMPATHETIC/NEUTRAL), toneConfidenceScore: int (1..5), decidedAt: String}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries within its 3-iteration budget.

- 1 Workflow DmWorkflow per messageId with two steps:
  * awaitSanitizedStep — polls DmEntity.getMessage every 1 s; on message.sanitized().isPresent()
    advances to replyStep. WorkflowSettings.stepTimeout 15 s.
  * replyStep — emits ReplyStarted, then calls componentClient.forAutonomousAgent(
    DmReplyAgent.class, "dm-agent-" + messageId).runSingleTask(
      TaskDef.instructions(formatBrandProfile(profile))
        .attachment("message.txt", sanitized.sanitizedText.getBytes())
    ) — returns a taskId, then forTask(taskId).result(DmTasks.REPLY_TO_MESSAGE) to fetch the
    reply. On success calls DmEntity.recordReply(reply). WorkflowSettings.stepTimeout 60 s
    with defaultStepRecovery maxRetries(2).failoverTo(DmWorkflow::error). error step
    transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity DmEntity (one per messageId). State DmMessage{messageId: String,
  inbound: Optional<InboundMessage>, sanitized: Optional<SanitizedMessage>,
  reply: Optional<DmReply>, status: MessageStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. MessageStatus enum: RECEIVED, SANITIZED, REPLYING,
  REPLIED, FAILED. Events: MessageReceived{inbound}, MessageSanitized{sanitized},
  ReplyStarted{}, ReplyRecorded{reply}, MessageFailed{reason}. Commands: receive,
  attachSanitized, markReplying, recordReply, fail, getMessage. emptyState() returns
  DmMessage.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to DmEntity events; on MessageReceived runs a
  regex+heuristic stripping pipeline (email addresses, phone numbers, URLs, government-
  identifier patterns, full-name heuristics) over rawText, computes the list of categories
  found, builds SanitizedMessage, then calls DmEntity.attachSanitized(sanitized). After
  attachSanitized lands, the same Consumer starts a DmWorkflow with id = "dm-" + messageId.

- 1 View DmView with row type DmRow (mirrors DmMessage minus inbound.rawText — the audit
  log keeps the raw; the view holds the sanitized form for the UI). Table updater consumes
  DmEntity events. ONE query getAllMessages: SELECT * AS messages FROM dm_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * DmEndpoint at /api with POST /messages (body {senderId, rawText, profileId}; mints
    messageId; calls DmEntity.receive; returns {messageId}), GET /messages (list from
    getAllMessages, sorted newest-first), GET /messages/{id} (one row), GET /messages/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- DmTasks.java declaring one Task<R> constant: REPLY_TO_MESSAGE = Task.name("Reply to
  message").description("Read the attached inbound DM and produce a DmReply matching the
  brand voice profile").resultConformsTo(DmReply.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records BrandProfile, InboundMessage, SanitizedMessage, DmReply, ToneLabel,
  DmMessage, MessageStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  DmReply, loads the matching BrandProfile from a in-memory registry, and runs the four
  checks listed in eval-matrix.yaml G1: no prohibited terms, no competitor names, within
  maxReplyChars, no out-of-scope commitment phrases. Returns Guardrail.reject(<structured-
  error>) on any failure; passes through on success.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9721 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/brand-profiles.jsonl with 3 seeded brand profiles:
  a Retail brand (prohibitedTerms: ["guarantee", "free", "unlimited"], maxReplyChars: 280),
  a Hospitality brand (prohibitedTerms: ["cheap", "complaint"], maxReplyChars: 320), and
  a SaaS brand (prohibitedTerms: ["hack", "workaround", "bug"], maxReplyChars: 250). Each
  has 2–3 competitor names.

- src/main/resources/sample-events/seed-messages.jsonl with 3 seeded inbound DMs:
  a product-availability inquiry, an order-status complaint, and a general brand question.
  Each contains at least one PII token (email or phone) so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous
  (the agent's reply is dispatched without human review in the baseline flow),
  oversight.human_on_loop = true (a moderator can audit the reply log),
  failure.failure_modes including "prohibited-term-in-reply", "pii-leakage-via-llm",
  "off-brand-tone", "out-of-scope-commitment", "competitor-mention";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/dm-reply-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: IG DM Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of message cards; right = selected-message detail with raw DM, PII category
  chips, sanitized preview, draft reply text, tone label, and tone-confidence score).
  Browser title exactly: <title>Akka Sample: IG DM Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct DmReply outputs per brand profile. Sets model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(messageId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    reply-to-message.json — 8 DmReply entries covering all four ToneLabel values.
      Each entry has a replyText paragraph matching the brand tone, a toneConfidenceScore
      (1..5), and a decidedAt timestamp. Plus 2 deliberately MALFORMED entries: one whose
      replyText contains a prohibited term from the Retail brand profile, and one whose
      replyText exceeds maxReplyChars — both rejected by the guardrail, exercising the retry
      path. The mock should select a malformed entry on the FIRST iteration of every 3rd
      message (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(messageId) helper makes per-message selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DmReplyAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DmTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (replyStep
  60 s, awaitSanitizedStep 15 s, error 5 s).
- Lesson 6: every nullable lifecycle field on the DmMessage row record is Optional<T>.
- Lesson 7: DmTasks.java with REPLY_TO_MESSAGE = Task.name(...).description(...)
  .resultConformsTo(DmReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9721 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (DmReplyAgent).
- The inbound message is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated replyStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
