# SPEC — routing-classifier-pattern

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Routing Workflow.
**One-line pitch:** A classifier agent reads an incoming customer message and routes it to one of three specialist agents that owns the reply end-to-end, with a before-invocation guardrail validating the route decision and a before-response guardrail screening every specialist draft.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which* specialist should own the reply, then the route decision is validated against an allowed-route registry before the chosen specialist is invoked. The specialist produces the final reply with no classifier involvement in the content. Two governance mechanisms are layered on top:

- A **before-agent-invocation guardrail** (`RouteGuardrail`) validates the classifier's `RouteDecision` against an allowed-route set before any specialist agent is called. If the proposed route is not in the registry, or if the classifier's confidence is below the minimum threshold, the guardrail rejects the decision and the workflow transitions to `ROUTE_BLOCKED` without invoking any specialist. This prevents the classifier from routing to a non-existent or wrong-mode specialist.
- A **before-agent-response guardrail** (`ReplyGuardrail`) checks the specialist's draft reply against a content-policy rubric before publishing. It blocks replies that promise timelines outside the stated policy, claim refund amounts the system cannot verify, or echo redacted tokens. A blocked draft lands in `REPLY_BLOCKED` for operator review.

The pattern is a fan-out-of-one: `RoutingWorkflow` branches on the validated route, exactly one specialist is invoked per message, and the others see no traffic.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live message list. Every message displays its route chip, status pill, and (if published) the final reply.
2. `MessageSimulator` (TimedAction) ticks every 30 s and inserts a new canned message from `sample-events/customer-messages.jsonl` into `MessageQueue`.
3. For each new message: `RoutingWorkflow` is started by the `MessageQueue` subscriber (`MessageIngestor` Consumer).
4. The workflow calls `RoutingClassifier`, gets a `RouteDecision { route, confidence, reason }`, and emits `MessageClassified` on the `MessageEntity`.
5. The workflow calls `RouteGuardrail` with the `RouteDecision`. The guardrail returns `RouteVerdict { approved, rejectionReason }`.
   - If `approved=false`: workflow emits `MessageRouteBlocked`; terminates in `ROUTE_BLOCKED`.
   - If `approved=true`: continue.
6. Branch on `route`:
   - `GENERAL` → workflow invokes `GeneralAgent` with the `REPLY` task.
   - `REFUND` → workflow invokes `RefundAgent` with the `REPLY` task.
   - `TECHNICAL` → workflow invokes `TechnicalAgent` with the `REPLY` task.
   - `UNROUTABLE` → workflow emits `MessageAbandoned`; terminates in `ABANDONED`.
7. The specialist returns a `Reply`. The workflow calls `ReplyGuardrail` with the `Reply`. If `allowed=true`, `ReplyPublished` is emitted (terminal `PUBLISHED`). If `allowed=false`, `ReplyBlocked` is emitted (terminal `REPLY_BLOCKED`) with the violation list.
8. The user can click any message card and see the raw message, the route decision, the route verdict, the chosen specialist, the draft reply, the guardrail verdict, and the published reply.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MessageSimulator` | `TimedAction` | Drips simulated customer messages into `MessageQueue` every 30 s. | scheduler | `MessageQueue` |
| `MessageQueue` | `EventSourcedEntity` | Append-only audit log of every inbound message (`InboundMessageReceived`). | `MessageSimulator`, `RoutingEndpoint` | `MessageIngestor` |
| `MessageIngestor` | `Consumer` | Subscribes to `MessageQueue` events; registers a `MessageEntity` and starts a `RoutingWorkflow`. | `MessageQueue` events | `MessageEntity`, `RoutingWorkflow` |
| `RoutingClassifier` | `Agent` (typed, not autonomous) | Classifies the raw message into `GENERAL` / `REFUND` / `TECHNICAL` / `UNROUTABLE` with confidence + reason. | invoked by `RoutingWorkflow` | returns `RouteDecision` |
| `RouteGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: validates the `RouteDecision` against the allowed-route registry. Returns `RouteVerdict { approved, rejectionReason }`. | invoked by `RoutingWorkflow` | returns `RouteVerdict` |
| `GeneralAgent` | `AutonomousAgent` | Owns the `REPLY` task for general enquiries. Returns typed `Reply`. | invoked by `RoutingWorkflow` | returns `Reply` |
| `RefundAgent` | `AutonomousAgent` | Owns the `REPLY` task for refund and billing requests. Returns typed `Reply`. | invoked by `RoutingWorkflow` | returns `Reply` |
| `TechnicalAgent` | `AutonomousAgent` | Owns the `REPLY` task for technical and integration questions. Returns typed `Reply`. | invoked by `RoutingWorkflow` | returns `Reply` |
| `ReplyGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `Reply` against the content-policy rubric. Returns `ReplyVerdict { allowed, violations, rubricVersion }`. | invoked by `RoutingWorkflow` | returns `ReplyVerdict` |
| `RoutingWorkflow` | `Workflow` | Per-message orchestration: classify → validate-route → branch → reply → screen → publish. | `MessageIngestor` (start) | `MessageEntity` |
| `MessageEntity` | `EventSourcedEntity` | Per-message lifecycle. | `RoutingWorkflow`, `MessageIngestor` | `MessageView` |
| `MessageView` | `View` | Read-model row per message. | `MessageEntity` events | `RoutingEndpoint` |
| `RoutingEndpoint` | `HttpEndpoint` | `/api/messages/*` — list, get, manual submit, operator unblock, SSE; `/api/metadata/*`. | — | `MessageView`, `MessageEntity`, `MessageQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingMessage(
    String messageId,
    String channel,            // "email" | "chat" | "web-form" | "phone-transcript"
    String subject,
    String body,
    Instant receivedAt
) {}

enum MessageRoute { GENERAL, REFUND, TECHNICAL, UNROUTABLE }

record RouteDecision(
    MessageRoute route,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

record RouteVerdict(
    boolean approved,
    String rejectionReason     // null when approved
) {}

enum ReplyAction { INFO_PROVIDED, REFUND_INITIATED, ARTICLE_LINKED, FOLLOW_UP_SCHEDULED, ESCALATED }

record Reply(
    String replySubject,
    String replyBody,
    ReplyAction action,
    String specialistTag,      // "general" | "refund" | "technical"
    Instant repliedAt
) {}

record ReplyVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record Message(
    String messageId,
    IncomingMessage incoming,
    Optional<RouteDecision> routeDecision,
    Optional<RouteVerdict> routeVerdict,
    Optional<Reply> reply,
    Optional<ReplyVerdict> replyVerdict,
    Optional<String> abandonReason,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED,
    CLASSIFIED,
    ROUTE_BLOCKED,
    ROUTED_GENERAL,
    ROUTED_REFUND,
    ROUTED_TECHNICAL,
    REPLY_DRAFTED,
    REPLY_BLOCKED,
    PUBLISHED,
    ABANDONED
}
```

Events on `MessageEntity`: `MessageRegistered`, `MessageClassified`, `MessageRouteBlocked`, `MessageRouted`, `ReplyDrafted`, `ReplyVerdictAttached`, `ReplyPublished`, `ReplyBlocked`, `MessageAbandoned`.

Events on `MessageQueue`: `InboundMessageReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/messages` — list all messages (newest-first), optional `?route=GENERAL|REFUND|TECHNICAL|UNROUTABLE&status=…` filtered client-side.
- `GET /api/messages/{id}` — one message.
- `POST /api/messages` — manually submit a message (body `IncomingMessage` minus `messageId` and `receivedAt`); server assigns both.
- `POST /api/messages/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `REPLY_BLOCKED` to `PUBLISHED` if the operator chooses to publish the screened draft.
- `GET /api/messages/sse` — Server-Sent Events for every message change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Routing Workflow</title>`.

The App UI tab is a three-pane layout: **left** is the message list (status pill + route chip), **centre** is the selected message's raw content, route decision, and route verdict, **right** is the chosen specialist's draft reply + content guardrail verdict + published reply (or violations + Unblock button when `REPLY_BLOCKED`; "Route blocked" notice when `ROUTE_BLOCKED`; "Abandoned" notice when `ABANDONED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail** on `RoutingClassifier`'s output: `RouteGuardrail` checks every `RouteDecision` against the allowed-route registry. Blocking — an unapproved route puts the message in `ROUTE_BLOCKED` with the rejection reason captured; no specialist sees the message.
- **G2 — before-agent-response guardrail** on `GeneralAgent`, `RefundAgent`, and `TechnicalAgent`: `ReplyGuardrail` checks every draft `Reply` against the content-policy rubric (no invented refund dates, no amounts outside verified policy, no legal or medical advice, no echoing of original PII). Blocking — a violation puts the message in `REPLY_BLOCKED` for operator review.

## 9. Agent prompts

- `RoutingClassifier` → `prompts/routing-classifier.md`. Typed classifier; returns one of `GENERAL`, `REFUND`, `TECHNICAL`, `UNROUTABLE`; defaults to `UNROUTABLE` under ambiguity.
- `RouteGuardrail` → `prompts/route-guardrail.md`. Validates route against the allowed-route registry and confidence floor. Returns `RouteVerdict { approved, rejectionReason }`.
- `GeneralAgent` → `prompts/general-agent.md`. Owns the `REPLY` task for general enquiries (account questions, feature questions, policy questions outside billing/technical). Never makes billing or technical claims.
- `RefundAgent` → `prompts/refund-agent.md`. Owns the `REPLY` task for refund and billing requests. Never invents refund amounts or timelines; sets `action=ESCALATED` when outside authority.
- `TechnicalAgent` → `prompts/technical-agent.md`. Owns the `REPLY` task for technical and integration questions. Cites a help-centre article id when applicable; never invents one.
- `ReplyGuardrail` → `prompts/reply-guardrail.md`. Returns a `ReplyVerdict { allowed, violations, rubricVersion }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a general-enquiry message → classified `GENERAL` → route guardrail approves → `GeneralAgent` produces a `Reply` → reply guardrail passes → published.
2. **J2** — Simulator drips a refund message → classified `REFUND` → `RefundAgent` resolves it → published.
3. **J3** — Simulator drips a technical message → classified `TECHNICAL` → `TechnicalAgent` resolves it → published.
4. **J4** — An ambiguous one-liner is classified `UNROUTABLE` and the workflow terminates in `ABANDONED` without invoking any specialist.
5. **J5** — A `RouteDecision` with a disallowed route (e.g. route name not in the registry) is rejected by `RouteGuardrail`; message lands in `ROUTE_BLOCKED` with the rejection reason; no specialist is invoked.
6. **J6** — A draft that promises a specific refund date (outside policy) is blocked by `ReplyGuardrail`; message lands in `REPLY_BLOCKED` with the violation listed; operator can unblock.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named routing-classifier-pattern demonstrating the
handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real messaging
integration).
Maven group io.akka.samples. Maven artifact
handoff-routing-cx-support-routing-classifier-pattern. Java package
io.akka.samples.routingworkflow. Akka 3.6.0. HTTP port 9633.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RoutingClassifier — classifier. System
  prompt loaded from prompts/routing-classifier.md. Input:
  IncomingMessage{messageId, channel, subject, body, receivedAt}. Output:
  RouteDecision{route: MessageRoute (GENERAL/REFUND/TECHNICAL/UNROUTABLE),
  confidence: "high"|"medium"|"low", reason: String}. Defaults to
  UNROUTABLE under uncertainty.

- 1 Agent (typed) RouteGuardrail — before-agent-invocation guardrail.
  System prompt from prompts/route-guardrail.md. Input: RouteDecision.
  Output: RouteVerdict{approved: boolean, rejectionReason: String (null
  when approved)}. Checks that route is in {GENERAL, REFUND, TECHNICAL}
  and confidence >= "medium". If route is UNROUTABLE or confidence is
  "low", approved=false.

- 1 AutonomousAgent GeneralAgent — definition() with
  capability(TaskAcceptance.of(REPLY).maxIterationsPerTask(3)). System
  prompt from prompts/general-agent.md. Input: IncomingMessage +
  RouteDecision. Output: Reply{replySubject, replyBody, action:
  ReplyAction, specialistTag = "general", repliedAt}. Does not make
  billing or technical claims.

- 1 AutonomousAgent RefundAgent — definition() with
  capability(TaskAcceptance.of(REPLY).maxIterationsPerTask(3)). System
  prompt from prompts/refund-agent.md. Same input shape; specialistTag =
  "refund". Never invents refund amounts or timelines; sets
  action=ESCALATED with a reason when outside authority.

- 1 AutonomousAgent TechnicalAgent — definition() with
  capability(TaskAcceptance.of(REPLY).maxIterationsPerTask(3)). System
  prompt from prompts/technical-agent.md. Same input shape; specialistTag
  = "technical". Cites help-centre article ids; never invents one.

- 1 Agent (typed) ReplyGuardrail — before-agent-response guardrail. System
  prompt from prompts/reply-guardrail.md. Input: IncomingMessage + Reply.
  Output: ReplyVerdict{allowed: boolean, violations: List<String>,
  rubricVersion: String}. Used by RoutingWorkflow before publishing;
  blocking.

- 1 Workflow RoutingWorkflow per messageId. Steps:
    classifyStep -> validateRouteStep -> {generalStep | refundStep |
                    technicalStep | abandonStep}
                 -> screenStep -> publishStep
  classifyStep calls componentClient.forAgent().inSession(messageId)
    .method(RoutingClassifier::classify).invoke(incoming). On success
    emits MessageClassified via MessageEntity.recordClassification.
  validateRouteStep calls forAgent(...).method(RouteGuardrail::validate)
    .invoke(decision). On verdict.approved=false emit
    MessageRouteBlocked(rejectionReason); terminates in ROUTE_BLOCKED.
  On approved=true, branch on decision.route:
    GENERAL -> proceed to generalStep (emits MessageRouted{GENERAL})
    REFUND -> proceed to refundStep (emits MessageRouted{REFUND})
    TECHNICAL -> proceed to technicalStep (emits MessageRouted{TECHNICAL})
    UNROUTABLE -> abandonStep (emits MessageAbandoned; terminates).
  generalStep / refundStep / technicalStep call
    forAutonomousAgent(<Agent>.class, messageId)
    .runSingleTask(TaskDef.instructions(buildPrompt(incoming, decision)))
    returning a taskId, then forTask(taskId).result(RoutingTasks.REPLY)
    to block on the typed Reply. On success emits ReplyDrafted.
  screenStep calls forAgent(...).method(ReplyGuardrail::screen).invoke(
    incoming, draft). On verdict.allowed=true proceed to publishStep
    (emits ReplyPublished, terminal PUBLISHED). On verdict.allowed=false
    emit ReplyBlocked (terminal REPLY_BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on
    classifyStep and validateRouteStep and screenStep,
    stepTimeout(Duration.ofSeconds(60)) on generalStep, refundStep,
    technicalStep, and publishStep.
    defaultStepRecovery(maxRetries(2).failoverTo(RoutingWorkflow::error)).

- 2 EventSourcedEntities:
    * MessageQueue — append-only audit log. Command receive(IncomingMessage)
      emits InboundMessageReceived{incoming}. No mutable state beyond a
      counter; commands are idempotent on incoming.messageId.
    * MessageEntity (one per messageId) — full per-message lifecycle.
      State Message{messageId, incoming: IncomingMessage,
      Optional<RouteDecision> routeDecision, Optional<RouteVerdict>
      routeVerdict, Optional<Reply> reply, Optional<ReplyVerdict>
      replyVerdict, Optional<String> abandonReason, MessageStatus status,
      Instant createdAt, Optional<Instant> finishedAt}. MessageStatus
      enum: RECEIVED, CLASSIFIED, ROUTE_BLOCKED, ROUTED_GENERAL,
      ROUTED_REFUND, ROUTED_TECHNICAL, REPLY_DRAFTED, REPLY_BLOCKED,
      PUBLISHED, ABANDONED. Events: MessageRegistered, MessageClassified,
      MessageRouteBlocked, MessageRouted, ReplyDrafted,
      ReplyVerdictAttached, ReplyPublished, ReplyBlocked,
      MessageAbandoned. Commands: registerMessage, recordClassification,
      recordRouteBlocked, recordRouting, recordDraft, recordReplyVerdict,
      publish, blockReply, abandon, unblock, getMessage.
      emptyState() returns Message.initial("") with no commandContext()
      reference.

- 1 Consumer MessageIngestor subscribed to MessageQueue events; for each
  InboundMessageReceived calls MessageEntity.registerMessage for the
  messageId; then starts a RoutingWorkflow with messageId as the workflow
  id.

- 1 View MessageView with row type MessageRow (mirrors Message; uses
  Optional<T> for every nullable lifecycle field per Lesson 6). Table
  updater consumes MessageEntity events. ONE query getAllMessages SELECT *
  AS messages FROM message_view. No WHERE route or WHERE status filter
  (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction MessageSimulator — every 30s, reads next line from
  src/main/resources/sample-events/customer-messages.jsonl (loops at EOF)
  and calls MessageQueue.receive with a fresh messageId (UUID).

- 2 HttpEndpoints:
    * RoutingEndpoint at /api with GET /messages (list from
      MessageView.getAllMessages, filter client-side by ?route and ?status
      query params), GET /messages/{id}, POST /messages (body
      IncomingMessage minus messageId/receivedAt — server assigns),
      POST /messages/{id}/unblock (body {decidedBy, note} — operator
      override: publishes the blocked draft as PUBLISHED with an audit
      note), GET /messages/sse (serverSentEventsForView over
      getAllMessages), and three /api/metadata/{readme,risk-survey,
      eval-matrix} endpoints serving the YAML/MD files from
      src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
      static-resources/*.

Companion files:
- RoutingTasks.java declaring the task constant: REPLY (resultConformsTo
  Reply.class, description "Handle the customer message end-to-end and
  return a typed Reply").
- Domain records IncomingMessage, RouteDecision, RouteVerdict, Reply,
  ReplyVerdict, and the Message entity state.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9633 and the three model-provider
  blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/customer-messages.jsonl with 10 canned
  lines (3 GENERAL-flavoured, 3 REFUND-flavoured, 2 TECHNICAL-flavoured,
  1 UNROUTABLE/ambiguous one-liner, 1 designed to trip the reply
  guardrail with a specific-date refund promise).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the root-level files for the endpoint to serve
  from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail
  before-agent-invocation route-registry, G2 guardrail
  before-agent-response policy-rubric. Matching simplified_view list.
  No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with
  purpose.primary_function = customer-support,
  data.data_classes.pii = false (no PII sanitizer in this baseline),
  decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false,
  failure.failure_modes including "wrong-route-decision",
  "inappropriate-reply-content", "route-bypass",
  "refund-claim-fabrication"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/routing-classifier.md, prompts/route-guardrail.md,
  prompts/general-agent.md, prompts/refund-agent.md,
  prompts/technical-agent.md, prompts/reply-guardrail.md loaded as agent
  system prompts.
- README.md at the project root: title "Akka Sample: Routing Workflow",
  prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained
  file (no ui/, no npm). Five tabs matching the formal exemplar. App UI
  tab uses a three-column layout (left = message list with route chip +
  status pill; centre = raw message + route decision + route verdict;
  right = specialist draft + reply verdict + published reply or violations
  + Unblock button). Browser title exactly:
  <title>Akka Sample: Routing Workflow</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a
        project-local .akka-build.yaml; /akka:build sources the file
        before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at
        run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only
  the REFERENCE (env-var name, file path, secrets URI); the value lives
  in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by messageId so reruns are
  deterministic), and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    routing-classifier.json — 12 RouteDecision entries spanning GENERAL
      (account questions, feature questions, policy questions), REFUND
      (charge disputes, cancellation requests, billing errors), TECHNICAL
      (API errors, integration questions, configuration help), and
      UNROUTABLE (very short messages, off-topic, mixed). Confidence +
      a one-sentence reason on each.
    route-guardrail.json — 12 RouteVerdict entries. 10 with
      approved=true. 2 with approved=false (rejectionReason
      "route-confidence-below-threshold" and "route-not-in-registry").
      The mock should return approved=false for messageIds that produce
      low-confidence UNROUTABLE decisions.
    general-agent.json — 6 Reply entries: 4 with action INFO_PROVIDED,
      1 with action FOLLOW_UP_SCHEDULED, 1 designed to trip the reply
      guardrail (claims a specific legal right the system cannot verify).
    refund-agent.json — 6 Reply entries: 4 with action REFUND_INITIATED
      or INFO_PROVIDED, 1 with action ESCALATED, 1 designed to trip the
      reply guardrail (promises a specific refund-processing date outside
      policy).
    technical-agent.json — 6 Reply entries: 4 with INFO_PROVIDED or
      ARTICLE_LINKED, 1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED.
    reply-guardrail.json — 10 ReplyVerdict entries. 8 with allowed=true
      and empty violations. 2 with allowed=false and one violation each
      ("invented-refund-date", "unverifiable-legal-claim"). The mock
      should match the guardrail-tripping entries above when called for
      the same messageId.
- A MockModelProvider.seedFor(messageId) helper makes per-message
  selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  GeneralAgent, RefundAgent, and TechnicalAgent all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings():
  classifyStep 20s, validateRouteStep 20s, screenStep 20s, generalStep /
  refundStep / technicalStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Message is Optional<T>.
  The MessageView row type uses the same Optional wrapping.
- (Lesson 7) RoutingTasks.java declares the REPLY Task<Reply> constant.
  All three specialists' definition().capability(TaskAcceptance.of(REPLY)
  ...) reference it.
- (Lesson 8) Model names verified against current lineup:
  claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9633 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal
  scroll.
- (Lesson 13) Integration tier label is "Runs out of the box".
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS
  overrides AND theme variables (state-diagram label colour white,
  edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The before-agent-invocation guardrail (RouteGuardrail) fires BEFORE any
  specialist agent is invoked — not after the specialist has already
  produced output. A rejected route never reaches any specialist.
- The before-agent-response guardrail (ReplyGuardrail) fires BEFORE
  ReplyPublished. A blocked draft never reaches the UI as published.
- No forbidden words in user-facing text: shape, minimal, smaller,
  complex, Akka SDK in narrative, T1/T2/T3/T4, deferred, use,
  use, marketing tone, competitor brand names.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
