# SPEC — slack-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SlackAssistant.
**One-line pitch:** A Slack-bot agent listens for messages in channels, sanitizes the message context to strip any embedded secrets, then calls one AI agent that returns a structured reply — action type, response text, and optional runbook reference — which is posted back to the originating thread.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `ChannelAssistantAgent` (AutonomousAgent) carries the entire decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **secret sanitizer** runs inside a Consumer between the raw message event and the agent call — so the model never sees API tokens, webhook URLs, or credentials that ops teams routinely paste into channels.
- A **before-agent-response guardrail** validates the agent's reply on every turn: well-formed JSON, action type in the allowed set, no secret-pattern strings in the response text. A reply that fails any check triggers a retry inside the same task.

The blueprint shows that even a conversational ops assistant needs deliberate controls — secret leakage from Slack history and hallucinated action types are real failure modes for a channel bot.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a seeded Slack message from the **Message** dropdown (or pastes a raw message payload into the textarea), sets the **Channel** and **User** fields, and clicks **Send to agent**.
2. The UI POSTs to `/api/messages` and receives a `messageId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `SANITIZED` — the scrubbed context is visible in the card detail, with a small list of secret categories that were found and removed.
4. Within ~10–30 s, `replyStep` completes. The card transitions to `REPLYING` then `REPLY_RECORDED`. The reply appears: the action type badge (`ANSWER` / `ESCALATE` / `RUNBOOK_REF` / `CLARIFY`), the response text, and an optional runbook URL.
5. Within ~1 s the `auditStep` finishes. The card shows a **guardrail-pass count** chip (how many candidate responses were accepted vs. retried).
6. The user can send another message; the live list keeps the full history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MessageEndpoint` | `HttpEndpoint` | `/api/messages/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `MessageEntity`, `MessageView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `MessageEntity` | `EventSourcedEntity` | Per-message lifecycle: received → sanitized → replying → reply recorded → audited. Source of truth. | `MessageEndpoint`, `SecretSanitizer`, `MessageWorkflow` | `MessageView` |
| `SecretSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; scrubs secrets; calls `MessageEntity.attachSanitized`. | `MessageEntity` events | `MessageEntity` |
| `MessageWorkflow` | `Workflow` | One workflow per message. Steps: `awaitSanitizedStep` → `replyStep` → `auditStep`. | started by `SecretSanitizer` once sanitized event lands | `ChannelAssistantAgent`, `MessageEntity` |
| `ChannelAssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the channel routing context in the task definition and the sanitized message window as a task attachment; returns `AssistantReply`. | invoked by `MessageWorkflow` | returns reply |
| `MessageView` | `View` | Read model: one row per message for the UI. | `MessageEntity` events | `MessageEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum ActionType { ANSWER, ESCALATE, RUNBOOK_REF, CLARIFY }

record ChannelContext(
    String channelId,
    String channelName,
    String triggerUserId,
    String triggerText,
    Instant receivedAt
) {}

record SanitizedContext(
    String scrubbedText,
    List<String> secretCategoriesFound
) {}

record AssistantReply(
    ActionType actionType,
    String responseText,
    Optional<String> runbookUrl,
    int guardrailIterations,
    Instant repliedAt
) {}

record AuditRecord(
    int candidateCount,
    int rejectedCount,
    String outcome,
    Instant auditedAt
) {}

record Message(
    String messageId,
    Optional<ChannelContext> context,
    Optional<SanitizedContext> sanitized,
    Optional<AssistantReply> reply,
    Optional<AuditRecord> audit,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED, SANITIZED, REPLYING, REPLY_RECORDED, AUDITED, FAILED
}
```

Events on `MessageEntity`: `MessageReceived`, `MessageSanitized`, `ReplyStarted`, `ReplyRecorded`, `AuditCompleted`, `MessageFailed`.

Every nullable lifecycle field on the `Message` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/messages` — body `{ channelId, channelName, triggerUserId, triggerText }` → `{ messageId }`.
- `GET /api/messages` — list all messages, newest-first.
- `GET /api/messages/{id}` — one message.
- `GET /api/messages/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Slack Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of inbound messages (status pill + action badge + age) and a right pane with the selected message's detail — channel context, sanitized text preview, reply text and action badge, and guardrail iteration count.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ChannelAssistantAgent`. Asserts the candidate response is well-formed `AssistantReply` JSON, the `actionType` is in `{ANSWER, ESCALATE, RUNBOOK_REF, CLARIFY}`, the `responseText` contains no string that matches known secret patterns (xoxb-, xoxp-, ghp_, glpat-, AKIA, Bearer [A-Za-z0-9+/]{20,}), and `responseText` is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **S1 — secret sanitizer** (`secret`, applied inside `SecretSanitizer` Consumer): strips Slack bot tokens, Slack user tokens, GitHub personal-access tokens, GitLab tokens, AWS access-key IDs, generic Bearer tokens, and webhook URLs from the inbound message text and channel history window before any LLM call. Records which categories were found.

## 9. Agent prompts

- `ChannelAssistantAgent` → `prompts/channel-assistant.md`. The single decision-making LLM. System prompt instructs it to read the attached message context, classify the request, and return a single `AssistantReply` with an appropriate action type and a concise response.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends the seeded "deploy status" message; within 30 s the reply appears with action type `ANSWER` and non-empty `responseText`.
2. **J2** — The agent's first response on a message (mock LLM path) contains an `xoxb-` token in `responseText` — the `before-agent-response` guardrail rejects it; the second iteration produces a clean reply; the channel never receives the rejected response.
3. **J3** — A message context window containing `xoxb-T...` and `ghp_...` strings is submitted; the task attachment seen by the agent shows only `[REDACTED-SLACK-TOKEN]` and `[REDACTED-GH-PAT]`; `context.triggerText` on the entity retains the originals for audit.
4. **J4** — A message classified as `ESCALATE` shows the `ESCALATE` action badge in the live list and the detail pane.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named slack-assistant demonstrating the single-agent × ops-automation cell.
Requires a Slack Bot Token at runtime (not at build time). Maven group io.akka.samples.
Maven artifact single-agent-ops-automation-slack-assistant. Java package
io.akka.samples.slackassistant. Akka 3.6.0. HTTP port 9945.

Components to wire (exactly):

- 1 AutonomousAgent ChannelAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/channel-assistant.md>) and
  .capability(TaskAcceptance.of(MessageTasks.REPLY_TO_MESSAGE).maxIterationsPerTask(3)).
  The task receives channel routing context in its instruction text and the sanitized message
  window as a task ATTACHMENT (NOT as inline prompt text —
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  AssistantReply{actionType: ActionType (ANSWER/ESCALATE/RUNBOOK_REF/CLARIFY),
  responseText: String, runbookUrl: Optional<String>, guardrailIterations: int,
  repliedAt: Instant}. The agent is configured with a before-agent-response guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow MessageWorkflow per messageId with three steps:
  * awaitSanitizedStep — polls MessageEntity.getMessage every 1s; on
    message.sanitized().isPresent() advances to replyStep.
    WorkflowSettings.stepTimeout 15s.
  * replyStep — emits ReplyStarted, then calls componentClient.forAutonomousAgent(
    ChannelAssistantAgent.class, "assistant-" + messageId).runSingleTask(
      TaskDef.instructions(formatContext(message.context))
        .attachment("context.txt", message.sanitized.scrubbedText.getBytes())
    ) — returns a taskId, then forTask(taskId).result(MessageTasks.REPLY_TO_MESSAGE)
    to fetch the reply. On success calls MessageEntity.recordReply(reply).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(MessageWorkflow::error).
  * auditStep — runs a deterministic AuditRecorder (NOT an LLM call) that counts
    guardrail candidate responses and accepted vs. rejected counts from the task result
    metadata, builds AuditRecord, and calls MessageEntity.recordAudit(audit).
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity MessageEntity (one per messageId). State
  Message{messageId: String, context: Optional<ChannelContext>,
  sanitized: Optional<SanitizedContext>, reply: Optional<AssistantReply>,
  audit: Optional<AuditRecord>, status: MessageStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. MessageStatus enum: RECEIVED, SANITIZED, REPLYING,
  REPLY_RECORDED, AUDITED, FAILED. Events: MessageReceived{context}, MessageSanitized{sanitized},
  ReplyStarted{}, ReplyRecorded{reply}, AuditCompleted{audit}, MessageFailed{reason}.
  Commands: receive, attachSanitized, markReplying, recordReply, recordAudit, fail,
  getMessage. emptyState() returns Message.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer SecretSanitizer subscribed to MessageEntity events; on MessageReceived runs
  a regex scrubbing pipeline (Slack bot tokens xoxb-, Slack user tokens xoxp-, GitHub PAT
  ghp_, GitLab tokens glpat-, AWS access-key IDs AKIA[0-9A-Z]{16}, generic Bearer tokens,
  webhook URLs containing /hooks/ or /services/) over context.triggerText, computes the list
  of secret categories found, builds SanitizedContext, then calls
  MessageEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a MessageWorkflow with id = "msg-" + messageId.

- 1 View MessageView with row type MessageRow (mirrors Message minus context.triggerText
  raw form — the audit log keeps the raw; the view holds the sanitized form for the UI).
  Table updater consumes MessageEntity events. ONE query getAllMessages:
  SELECT * AS messages FROM message_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * MessageEndpoint at /api with POST /messages (body
    {channelId, channelName, triggerUserId, triggerText}; mints messageId; calls
    MessageEntity.receive; returns {messageId}), GET /messages (list from getAllMessages,
    sorted newest-first), GET /messages/{id} (one row), GET /messages/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- MessageTasks.java declaring one Task<R> constant: REPLY_TO_MESSAGE = Task.name("Reply
  to message").description("Read the attached sanitized message context and return an
  AssistantReply with an appropriate action type and response text")
  .resultConformsTo(AssistantReply.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ChannelContext, SanitizedContext, AssistantReply, AuditRecord, Message,
  MessageStatus, ActionType.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AssistantReply from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- AuditRecorder.java — pure deterministic logic (no LLM). Inputs: AssistantReply task
  result metadata (candidate count, rejected count). Outputs: AuditRecord. Logic documented
  in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9945 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ChannelAssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs. A separate config key
  akka.javasdk.slack.bot-token = ${?SLACK_BOT_TOKEN} holds the Slack credential reference.

- src/main/resources/sample-events/messages.jsonl with 5 seeded inbound Slack messages:
  a "deploy status" query, an "incident triage" request, a "runbook lookup" request, a
  "help with alerts" message, and a "scheduled maintenance window" announcement. Two messages
  contain embedded secrets (a fake xoxb- token and a fake ghp_ token) so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root pre-filled for ops-automation domain; deployer-specific
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/channel-assistant.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Slack Assistant", prerequisites
  (including Slack Bot Token sourcing), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of message cards; right = selected-message detail with channel context, sanitized
  text preview, reply text and action badge, and guardrail iteration count).
  Browser title exactly: <title>Akka Sample: Slack Assistant</title>. No subtitle on the
  Overview tab.

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
- Also ask how to source SLACK_BOT_TOKEN using the same five options. If the user selects
  Mock LLM (option a), the Slack posting step uses a MockSlackClient that logs the reply to
  stdout rather than calling the Slack API.
- NEVER write any key or token value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured
  credential material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(messageId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    reply-to-message.json — 8 AssistantReply entries covering all four ActionType values.
      Each entry has a non-empty responseText appropriate to the matching seeded message.
      Two ANSWER entries, two ESCALATE entries (one with a mock oncall reference), two
      RUNBOOK_REF entries with fake runbook URLs, one CLARIFY entry. Plus 2 deliberately
      MALFORMED entries (one with an actionType value not in the enum; one with a responseText
      containing "xoxb-FAKESLACKTOKEN" that the guardrail catches). The mock selects a
      malformed entry on the FIRST iteration of every 3rd message (modulo seed) so J2 is
      reproducible.
- A MockModelProvider.seedFor(messageId) helper makes per-message selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChannelAssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion MessageTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (replyStep
  60s, awaitSanitizedStep 15s, auditStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Message row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: MessageTasks.java with REPLY_TO_MESSAGE = Task.name(...).description(...)
  .resultConformsTo(AssistantReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9945 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Requires Slack Bot Token" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key and token sourcing — NEVER write any value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ChannelAssistantAgent).
  The audit recorder (AuditRecorder.java) is deterministic and does NOT make an LLM call.
- The message context is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated replyStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no credential export block.
  Per Lesson 25, /akka:specify handles key sourcing during generation.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key or Slack token (the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
