# SPEC — line-personal-assistant

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** LINE Personal Assistant.
**One-line pitch:** Send a LINE message; a planner agent interprets intent, routes to the right integration executor (Google Calendar or Gmail), applies a pre-call guardrail and an optional confirm-before-send gate, then replies directly in the LINE thread.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with two integration executors. The PlannerAgent reads each inbound LINE message and produces an `ActionPlan { intent, action, parameters }`. The workflow routes that plan to the matching executor agent — `CalendarExecutorAgent` for scheduling intents, `GmailExecutorAgent` for email intents — then posts the result back to the LINE thread.

The blueprint demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that checks every outbound write action — calendar create, email send — against scope rules and an email allow-list before any API call is made,
- a **human-in-the-loop confirmation gate** (application-tier HITL) that intercepts outbound email requests, sends a LINE confirmation message back to the user, and waits for explicit approval before dispatching the Gmail executor.

## 3. User-facing flows

The user sends a natural-language LINE message. The LINE webhook delivers it to the service.

1. The service records the inbound message to `MessageQueue` and the system creates a `Conversation` record in `PLANNING`.
2. The workflow asks `PlannerAgent` to produce an `ActionPlan`. The plan carries an `IntentKind` (`CREATE_EVENT`, `LIST_EVENTS`, `SEND_EMAIL`, `CLARIFY`) and typed `ActionParameters`.
3. The **before-tool-call guardrail** vets the plan. On rejection, the workflow records `ActionBlocked` and the system replies to the LINE thread with a human-readable explanation.
4. If intent is `CREATE_EVENT` or `LIST_EVENTS`, the workflow calls `CalendarExecutorAgent`; on success the conversation moves to `COMPLETED`.
5. If intent is `SEND_EMAIL`, the workflow enters the **confirm-before-send gate**:
   - The service sends a LINE confirmation message: "You're about to email `<recipient>` — Subject: `<subject>`. Reply YES to confirm or NO to cancel."
   - `ConfirmationEntity` records the pending confirmation keyed by the LINE reply-token.
   - The workflow suspends in `AWAITING_CONFIRMATION`.
   - When the user replies YES, the workflow resumes and calls `GmailExecutorAgent`.
   - When the user replies NO, the workflow records `ActionCancelled` and the conversation moves to `CANCELLED`.
6. If intent is `CLARIFY`, the workflow posts a clarification question back to the LINE thread and moves the conversation to `AWAITING_CLARIFICATION`.
7. `StaleConversationMonitor` ticks every 60 s; conversations stuck in `AWAITING_CONFIRMATION` or `AWAITING_CLARIFICATION` for more than 10 minutes are marked `EXPIRED`.

A `MessageSimulator` (TimedAction) drips a sample LINE message every 90 s so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Reads the user's LINE message and current conversation context; produces an `ActionPlan`. | `ConversationWorkflow` | returns typed result to workflow |
| `CalendarExecutorAgent` | `AutonomousAgent` | Executes a Google Calendar create-event or list-events call given `CalendarActionParams`. Returns `ExecutionResult`. | `ConversationWorkflow` | — |
| `GmailExecutorAgent` | `AutonomousAgent` | Composes and sends a Gmail message given `GmailActionParams`. Returns `ExecutionResult`. | `ConversationWorkflow` | — |
| `ConversationWorkflow` | `Workflow` | Drives the plan → guard → (confirm?) → execute → reply loop plus clarify and cancel branches. | `LineWebhookEndpoint`, `MessageConsumer` | `ConversationEntity`, `ConfirmationEntity` |
| `ConversationEntity` | `EventSourcedEntity` | Holds the conversation lifecycle, plan, execution result, and reply history. | `ConversationWorkflow` | `ConversationView` |
| `ConfirmationEntity` | `EventSourcedEntity` | Holds a pending outbound-email confirmation keyed by `conversationId`. | `ConversationWorkflow`, `LineWebhookEndpoint` | `ConversationWorkflow` (resume) |
| `MessageQueue` | `EventSourcedEntity` | Audit log of inbound LINE webhook events. | `LineWebhookEndpoint`, `MessageSimulator` | `MessageConsumer` |
| `ConversationView` | `View` | List-of-conversations read model for the monitoring UI. | `ConversationEntity` events | `LineWebhookEndpoint` |
| `MessageConsumer` | `Consumer` | Subscribes to `MessageQueue` events; starts a `ConversationWorkflow` per inbound message. | `MessageQueue` events | `ConversationWorkflow` |
| `MessageSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/line-messages.jsonl` and enqueues it. | scheduler | `MessageQueue` |
| `StaleConversationMonitor` | `TimedAction` | Every 60 s, marks conversations stuck in `AWAITING_CONFIRMATION` or `AWAITING_CLARIFICATION` past 10 minutes as `EXPIRED`. | scheduler | `ConversationEntity` |
| `LineWebhookEndpoint` | `HttpEndpoint` | `/api/webhook` (LINE callbacks), `/api/conversations/*` (list, get, SSE), `/api/confirmations/{id}/reply` (manual confirm). | — | `MessageQueue`, `ConversationView`, `ConfirmationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record InboundMessage(
    String lineUserId,
    String replyToken,
    String text,
    Instant receivedAt
) {}

record ActionPlan(
    IntentKind intent,
    String reasoning,
    Optional<CalendarActionParams> calendarParams,
    Optional<GmailActionParams> gmailParams,
    Optional<String> clarificationText
) {}

record CalendarActionParams(
    CalendarAction action,
    String title,
    Optional<String> description,
    Optional<Instant> startTime,
    Optional<Instant> endTime,
    Optional<String> timeZone
) {}

record GmailActionParams(
    String recipientEmail,
    String subject,
    String body,
    Optional<String> ccEmail
) {}

record ExecutionResult(
    IntentKind intent,
    boolean ok,
    String summary,
    Optional<String> errorReason,
    Instant executedAt
) {}

record ConfirmationRequest(
    String conversationId,
    GmailActionParams params,
    Instant requestedAt,
    Optional<Instant> respondedAt,
    ConfirmationStatus status
) {}

record Conversation(
    String conversationId,
    String lineUserId,
    String originalText,
    ConversationStatus status,
    Optional<ActionPlan> plan,
    Optional<ExecutionResult> result,
    Optional<String> blockerReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum IntentKind { CREATE_EVENT, LIST_EVENTS, SEND_EMAIL, CLARIFY }
enum CalendarAction { CREATE, LIST }
enum ConversationStatus {
    PLANNING, AWAITING_CONFIRMATION, AWAITING_CLARIFICATION,
    EXECUTING, COMPLETED, CANCELLED, BLOCKED, EXPIRED, FAILED
}
enum ConfirmationStatus { PENDING, APPROVED, DENIED, EXPIRED }
```

### Events (`ConversationEntity`)

`ConversationCreated`, `PlanProduced`, `ActionBlocked`, `ConfirmationRequested`, `ConfirmationApproved`, `ConfirmationDenied`, `ActionExecuted`, `ClarificationRequested`, `ActionCancelled`, `ConversationExpired`, `ConversationFailed`.

### Events (`ConfirmationEntity`)

`ConfirmationOpened`, `ConfirmationResolved { approved: boolean }`, `ConfirmationTimedOut`.

### Events (`MessageQueue`)

`MessageEnqueued { conversationId, lineUserId, text, receivedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/webhook` — LINE Messaging API callback (signature-verified). Dispatches inbound message events to `MessageQueue`. `200 {}`.
- `GET /api/conversations` — list all conversations. Optional `?status=...`.
- `GET /api/conversations/{id}` — one conversation (full plan + result).
- `GET /api/conversations/sse` — server-sent events stream of every conversation change.
- `POST /api/confirmations/{id}/reply` — body `{ approved: boolean }` → `200`. Resolves a pending confirmation (for testing; in production the LINE reply drives this).
- `GET /api/confirmations/{id}` — `{ conversationId, status, params }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "LINE Personal Assistant"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — simulated LINE chat pane showing conversation cards with status, plan details, confirmation gate state, and execution result. Operator controls for manually resolving confirmations.

Browser title: `<title>Akka Sample: LINE Personal Assistant</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `PlannerAgent`): every `ActionPlan` whose intent is `CREATE_EVENT`, `LIST_EVENTS`, or `SEND_EMAIL` is checked against (a) the integration scope allow-list (calendar writes require the calendar scope; email sends require the mail scope), (b) a recipient allow-list for `SEND_EMAIL` (deny-list of external domains configurable in `application.conf`), (c) a parameter completeness check (calendar create must have `startTime` and `endTime`; email must have `recipientEmail` and `subject`). Blocking. Failure → `ActionBlocked` event + LINE reply with explanation.
- **H1 — confirm-before-send HITL** (`hitl`, flavor `application`): outbound email is never dispatched without explicit user confirmation within the LINE thread. The workflow emits `ConfirmationRequested`, parks in `AWAITING_CONFIRMATION`, and only calls `GmailExecutorAgent` after `ConfirmationApproved`. `StaleConversationMonitor` expires unresponded confirmations after 10 minutes.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Reads inbound LINE text; produces `ActionPlan`.
- `CalendarExecutorAgent` → `prompts/calendar-executor.md`. Executes calendar API calls.
- `GmailExecutorAgent` → `prompts/gmail-executor.md`. Composes and sends email.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Send "Schedule a team sync with Alice tomorrow at 3 PM for 1 hour." Conversation progresses `PLANNING → EXECUTING → COMPLETED`. LINE reply contains event title and time. Guardrail passes.
2. **J2** — Send "Email the project update to bob@example.com." Guardrail passes. Conversation moves to `AWAITING_CONFIRMATION`. User replies YES. Conversation moves to `EXECUTING → COMPLETED`. Email sent; LINE reply confirms.
3. **J3** — Send "Email the project report to external-unknown@rival.io." Guardrail blocks the dispatch; conversation moves to `BLOCKED`. LINE reply explains the recipient domain is not allowed.
4. **J4** — Send "Email Alice the agenda." Confirmation request is sent. Let 10 minutes pass without reply. `StaleConversationMonitor` marks the conversation `EXPIRED`. No email is sent; user receives an expiry notice in LINE.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named line-personal-assistant demonstrating the
planner-executor × planning-travel cell. Requires Google OAuth credentials
and a LINE channel access token (sourced via the standard key-sourcing flow).
Maven group io.akka.samples. Maven artifact
planner-executor-planning-travel-personal-assistant. Java package
io.akka.samples.lineassistantwithcalendarandgmail. Akka 3.6.0. HTTP port 9573.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PlannerAgent — definition() with capabilities:
      capability(TaskAcceptance.of(PLAN_ACTION).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN_ACTION returns ActionPlan.
  * CalendarExecutorAgent — capability(TaskAcceptance.of(EXECUTE_CALENDAR)
      .maxIterationsPerTask(2)).
    Prompt from prompts/calendar-executor.md. Returns ExecutionResult.
  * GmailExecutorAgent — capability(TaskAcceptance.of(EXECUTE_EMAIL)
      .maxIterationsPerTask(2)).
    Prompt from prompts/gmail-executor.md. Returns ExecutionResult.

- 1 Workflow ConversationWorkflow with steps:
  planStep -> guardStep -> routeStep ->
    [if SEND_EMAIL] confirmRequestStep -> awaitConfirmStep ->
    [if approved] executeStep -> replyStep -> completeStep / failStep.
    [if denied] cancelStep.
    [if CLARIFY] clarifyStep -> completeStep.
    [if guardrail blocked] blockStep.
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(45), executeStep ofSeconds(60),
    confirmRequestStep ofSeconds(20), replyStep ofSeconds(20).
    defaultStepRecovery(maxRetries(2).failoverTo(ConversationWorkflow::error)).
  guardStep runs ActionPlanGuardrail.vet(plan); on reject calls
    ConversationEntity.recordBlock(reason) and transitions to blockStep
    (LINE reply sent in blockStep, conversation ends BLOCKED).
  confirmRequestStep emits ConfirmationRequested on ConversationEntity and
    ConfirmationOpened on ConfirmationEntity. Sends LINE confirm message.
  awaitConfirmStep polls ConfirmationEntity.get every 3 s up to 10 minutes.
    On APPROVED → executeStep. On DENIED → cancelStep. On EXPIRED → expireStep.
  executeStep switches on ActionPlan.intent to call either
    CalendarExecutorAgent or GmailExecutorAgent. Records result on
    ConversationEntity.
  replyStep sends LINE reply message with execution summary.

- 1 EventSourcedEntity ConversationEntity holding Conversation state.
  emptyState() returns Conversation.initial("", "", ""). Commands:
  createConversation, recordPlan, recordBlock, requestConfirmation,
  approveConfirmation, denyConfirmation, recordExecution, requestClarification,
  cancelAction, expireConversation, failConversation, getConversation.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity ConfirmationEntity keyed by conversationId. State
  ConfirmationRequest as in SPEC §5. Commands: openConfirmation,
  resolveConfirmation(approved), timeoutConfirmation, get.
  Events: ConfirmationOpened, ConfirmationResolved, ConfirmationTimedOut.

- 1 EventSourcedEntity MessageQueue with command enqueueMessage(conversationId,
  lineUserId, text, receivedAt) emitting MessageEnqueued.

- 1 View ConversationView with row type ConversationRow (mirror of Conversation
  minus full plan/result payload — truncate to status + key fields + last
  event; full fetch by id on click). ONE query getAllConversations SELECT *
  AS conversations FROM conversation_view. No WHERE filter — caller filters
  client-side (Lesson 2).

- 1 Consumer MessageConsumer subscribed to MessageQueue events; on
  MessageEnqueued starts a ConversationWorkflow with conversationId as the
  workflow id.

- 2 TimedActions:
  * MessageSimulator — every 90 s, reads next line from
    src/main/resources/sample-events/line-messages.jsonl and calls
    MessageQueue.enqueueMessage.
  * StaleConversationMonitor — every 60 s, queries ConversationView, filters
    conversations in AWAITING_CONFIRMATION or AWAITING_CLARIFICATION whose
    createdAt is older than 10 minutes, calls ConversationEntity.expireConversation.

- 2 HttpEndpoints:
  * LineWebhookEndpoint at /api with POST /webhook (LINE signature-verified
    callback), GET /conversations, GET /conversations/{id},
    GET /conversations/sse, POST /confirmations/{id}/reply,
    GET /confirmations/{id}, and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring one Task<R> constant: PLAN_ACTION
  (resultConformsTo ActionPlan).
- ExecutorTasks.java declaring two Task<R> constants: EXECUTE_CALENDAR,
  EXECUTE_EMAIL (both resultConformsTo ExecutionResult).
- Domain records as listed in SPEC §5.
- application/ActionPlanGuardrail.java — deterministic vetter. Reject if:
  (a) intent is SEND_EMAIL and recipientEmail domain is on the deny-list
      (configurable via akka.samples.email-deny-domains in application.conf,
      defaults to ["rival.io", "spam.test"]),
  (b) intent is SEND_EMAIL and recipientEmail is missing or blank,
  (c) intent is CREATE_EVENT and either startTime or endTime is absent,
  (d) intent is not a known IntentKind value.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9573 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also blocks for LINE_CHANNEL_ACCESS_TOKEN,
  GOOGLE_CALENDAR_OAUTH_TOKEN, GOOGLE_GMAIL_OAUTH_TOKEN — each read as
  ${?LINE_CHANNEL_ACCESS_TOKEN} etc. with fail-fast check in Bootstrap.java
  if any required integration credential is absent.
- src/main/resources/sample-events/line-messages.jsonl with 8 canned
  LINE messages spanning CREATE_EVENT, LIST_EVENTS, SEND_EMAIL, and CLARIFY
  intents.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, H1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root as specified in SPEC §8.
- prompts/planner.md, prompts/calendar-executor.md, prompts/gmail-executor.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: LINE Personal Assistant",
  one-line pitch, prerequisites (including the integration form's host-software
  requirement: Google OAuth credential + LINE channel access token), generate-
  the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section. NO
  "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (LINE chat simulation + confirmation gate pane + live
  conversation list with status pills and expand-on-click for plan,
  confirmation state, and result). Browser title exactly:
  <title>Akka Sample: LINE Personal Assistant</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently. Follow the same five-option flow for the integration
  credentials (LINE_CHANNEL_ACCESS_TOKEN, GOOGLE_CALENDAR_OAUTH_TOKEN,
  GOOGLE_GMAIL_OAUTH_TOKEN). A Mock integration mode is acceptable for
  the calendar and Gmail calls when the deployer selects option (a) during
  the key-sourcing prompt — generate a MockCalendarClient and MockGmailClient
  that return shape-correct responses from fixture files.
- NEVER write any key value to disk.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  executeStep, confirmRequestStep, replyStep.
- Lesson 6: Optional<T> for every nullable field.
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  ExecutorTasks.java.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9573 in application.conf.
- Lesson 11: source.platform never in user-facing surfaces.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Requires Google OAuth credential + LINE
  channel access token" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any file.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
