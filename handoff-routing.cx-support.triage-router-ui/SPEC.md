# SPEC — triage-router-ui

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Triage Agent (UX).
**One-line pitch:** A router agent classifies an inbound customer support message and hands the resolution off to a billing or product handler that owns it end-to-end, with a before-agent-response guardrail checking every specialist draft before it is published.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the task, then transfers the same task identity to a downstream specialist agent that produces the final output. The downstream agent is responsible for the whole resolution; the classifier does not narrate or summarise. One governance mechanism is layered on top:

- A **before-agent-response guardrail** runs on every specialist's draft reply. It checks the draft against a policy rubric (no invented refund amounts, no timelines outside the published SLA, no medical or legal advice) and blocks the draft from reaching the user when it violates. The ticket transitions to `BLOCKED` and an operator must either override or leave it.

The pattern is a textbook fan-out-of-one: the workflow branches on the classifier's category, and only the chosen handler is invoked. The other handler sees no traffic for that message.

This blueprint also serves as the canonical embedded-UI reference for the handoff-routing pattern — the App UI tab is the primary interactive surface.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live message list. Every message displays its category chip, status pill, and (if resolved) the published response.
2. `MessageSimulator` (TimedAction) ticks every 30 s and inserts a new canned message from `sample-events/support-messages.jsonl` into `MessageQueue`.
3. For each new message: `TriageWorkflow` is started, `RouterAgent` classifies the message, and the workflow branches on `category`.
4. The workflow calls `RouterAgent`, gets a `RouteDecision { category, confidence, reason }`, and emits `MessageRouted` on the entity.
5. Branch on `category`:
   - `BILLING` → workflow calls `BillingHandler` with the `RESOLVE` task and waits for the typed `HandlerReply` result.
   - `PRODUCT` → workflow calls `ProductHandler` with the same `RESOLVE` task.
   - `UNCLEAR` → workflow emits `MessageEscalated`; ends.
6. The handler's draft `HandlerReply` passes through the before-agent-response guardrail. If accepted, `ReplyPublished` is emitted (terminal `RESOLVED`). If rejected, `ReplyBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. The user can click any message card and see the original message, the routing decision, the chosen handler's draft, the guardrail verdict, and the published response (or blocked draft + violations + Unblock button).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MessageSimulator` | `TimedAction` | Drips simulated support messages into `MessageQueue` every 30 s. | scheduler | `MessageQueue` |
| `MessageQueue` | `EventSourcedEntity` | Append-only audit log of every inbound message (`InboundMessageReceived`). | `MessageSimulator`, `TriageEndpoint` | `TriageWorkflow` |
| `RouterAgent` | `Agent` (typed, not autonomous) | Classifies a `SupportMessage` into `BILLING` / `PRODUCT` / `UNCLEAR` with confidence + reason. | invoked by `TriageWorkflow` | returns `RouteDecision` |
| `BillingHandler` | `AutonomousAgent` | Owns the `RESOLVE` task for billing messages. Returns typed `HandlerReply`. | invoked by `TriageWorkflow` | returns `HandlerReply` |
| `ProductHandler` | `AutonomousAgent` | Owns the `RESOLVE` task for product messages. Returns typed `HandlerReply`. | invoked by `TriageWorkflow` | returns `HandlerReply` |
| `DraftGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `HandlerReply` against the policy rubric. Returns `GuardrailResult { allowed, violations }`. | invoked by `TriageWorkflow` | returns `GuardrailResult` |
| `TriageWorkflow` | `Workflow` | Per-message orchestration: route → resolve → guardrail → publish. | `MessageQueue` (start) | `MessageEntity` |
| `MessageEntity` | `EventSourcedEntity` | Per-message lifecycle. | `TriageWorkflow` | `MessageView` |
| `MessageView` | `View` | Read-model row per message. | `MessageEntity` events | `TriageEndpoint` |
| `TriageEndpoint` | `HttpEndpoint` | `/api/messages/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `MessageView`, `MessageEntity`, `MessageQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record SupportMessage(
    String messageId,
    String customerId,
    String channel,           // "email" | "chat" | "web-form"
    String subject,
    String body,
    Instant receivedAt
) {}

enum MessageCategory { BILLING, PRODUCT, UNCLEAR }

record RouteDecision(
    MessageCategory category,
    String confidence,        // "high" | "medium" | "low"
    String reason             // one short sentence
) {}

enum ReplyAction { REFUND_INITIATED, ARTICLE_LINKED, FOLLOW_UP_SCHEDULED, INFO_PROVIDED, ESCALATED }

record HandlerReply(
    String replySubject,
    String replyBody,
    ReplyAction action,
    String handlerTag,        // "billing" | "product"
    Instant repliedAt
) {}

record GuardrailResult(
    boolean allowed,
    List<String> violations,  // empty when allowed
    String rubricVersion
) {}

record Message(
    String messageId,
    SupportMessage incoming,
    Optional<RouteDecision> route,
    Optional<HandlerReply> reply,
    Optional<GuardrailResult> guardrail,
    Optional<String> escalationReason,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED,
    ROUTED_BILLING,
    ROUTED_PRODUCT,
    REPLY_DRAFTED,
    BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `MessageEntity`: `MessageRegistered`, `MessageRouted`, `ReplyDrafted`, `GuardrailResultAttached`, `ReplyPublished`, `ReplyBlocked`, `MessageEscalated`.

Events on `MessageQueue`: `InboundMessageReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/messages` — list all messages (newest-first), optional `?category=BILLING|PRODUCT|UNCLEAR&status=…` filtered client-side.
- `GET /api/messages/{id}` — one message.
- `POST /api/messages` — manually submit a message (body `SupportMessage` minus `messageId` and `receivedAt`); server assigns both.
- `POST /api/messages/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator chooses to publish the blocked draft.
- `GET /api/messages/sse` — Server-Sent Events for every message change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Triage Agent (UX)</title>`.

The App UI tab is a three-pane layout: **left** is the message list (status pill + category chip), **centre** is the selected message's body + routing decision, **right** is the chosen handler's draft + guardrail verdict + published response (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** on `BillingHandler` and `ProductHandler`: checks every draft `HandlerReply` against a rubric (no invented refund amounts, no specific timelines outside published SLA, no medical/legal/financial advice, no `[REDACTED]` echoes). Blocking — a violation puts the message in `BLOCKED` for human review.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. Typed classifier; returns one of `BILLING`, `PRODUCT`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `BillingHandler` → `prompts/billing-handler.md`. Owns the `RESOLVE` task for billing messages. Never invents refund amounts; routes outside its authority to `ESCALATED`.
- `ProductHandler` → `prompts/product-handler.md`. Owns the `RESOLVE` task for product messages. Cites a help-centre article id when one applies; never invents one.
- `DraftGuardrail` → `prompts/draft-guardrail.md`. Returns a `GuardrailResult { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a billing-flavoured message → routed `BILLING` → resolved by `BillingHandler` → guardrail passes → published.
2. **J2** — Simulator drips a product message → routed `PRODUCT` → resolved by `ProductHandler` → guardrail passes → published.
3. **J3** — An ambiguous message routes as `UNCLEAR` and lands in `ESCALATED` without any handler invocation.
4. **J4** — A draft that promises a 24-hour refund (outside policy) is blocked; the message lands in `BLOCKED` with the violation listed; the operator can either unblock or leave it.
5. **J5** — An operator unblocks a blocked message via the UI; the draft is published and the override note is preserved in the audit record.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named triage-router-ui demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real messaging integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-triage-router-ui.
Java package io.akka.samples.triageagentux. Akka 3.6.0. HTTP port 9573.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RouterAgent — classifier. System prompt loaded from
  prompts/router-agent.md. Input: SupportMessage{messageId, customerId, channel, subject,
  body, receivedAt}. Output: RouteDecision{category: MessageCategory
  (BILLING/PRODUCT/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent BillingHandler — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/billing-handler.md. Input:
  SupportMessage + RouteDecision. Output: HandlerReply{replySubject, replyBody,
  action: ReplyAction, handlerTag = "billing", repliedAt}. Never invents refund
  amounts; sets action=ESCALATED with a reason when outside authority.

- 1 AutonomousAgent ProductHandler — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/product-handler.md. Same
  input shape; handlerTag = "product".

- 1 Agent (typed) DraftGuardrail — typed rubric check. System prompt from
  prompts/draft-guardrail.md. Input: SupportMessage + HandlerReply. Output:
  GuardrailResult{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by TriageWorkflow before publishing; blocking.

- 1 Workflow TriageWorkflow per messageId. Steps:
    routeStep -> {billingStep | productStep | escalateStep}
              -> guardrailStep -> publishStep
  routeStep calls componentClient.forAgent().inSession(messageId).method(RouterAgent::route)
    .invoke(message). On success emits MessageRouted via MessageEntity.recordRoute.
    Branches on RouteDecision.category:
    BILLING -> proceed to billingStep (emits MessageRouted{BILLING})
    PRODUCT -> proceed to productStep (emits MessageRouted{PRODUCT})
    UNCLEAR -> escalateStep (emits MessageEscalated; terminates).
  billingStep / productStep call forAutonomousAgent(<Handler>.class, messageId)
    .runSingleTask(TaskDef.instructions(buildPrompt(message, route))) returning a taskId,
    then forTask(taskId).result(TriageTasks.RESOLVE) to block on the typed HandlerReply.
    On success emits ReplyDrafted.
  guardrailStep calls forAgent(...).method(DraftGuardrail::check).invoke(message, draft).
    On result.allowed=true proceed to publishStep (emits ReplyPublished, terminal RESOLVED).
    On result.allowed=false emit ReplyBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on routeStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on billingStep, productStep, and
    publishStep. defaultStepRecovery(maxRetries(2).failoverTo(TriageWorkflow::error)).

- 2 EventSourcedEntities:
    * MessageQueue — append-only audit log. Command receive(SupportMessage) emits
      InboundMessageReceived{message}. No mutable state beyond a counter; commands are
      idempotent on message.messageId.
    * MessageEntity (one per messageId) — full per-message lifecycle. State
      Message{messageId, incoming: SupportMessage, Optional<RouteDecision> route,
      Optional<HandlerReply> reply, Optional<GuardrailResult> guardrail,
      Optional<String> escalationReason, MessageStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. MessageStatus enum: RECEIVED, ROUTED_BILLING,
      ROUTED_PRODUCT, REPLY_DRAFTED, BLOCKED, RESOLVED, ESCALATED.
      Events: MessageRegistered, MessageRouted, ReplyDrafted, GuardrailResultAttached,
      ReplyPublished, ReplyBlocked, MessageEscalated. Commands: registerMessage,
      recordRoute, recordDraft, recordGuardrailResult, publish, block, escalate,
      unblock, getMessage. emptyState() returns Message.initial("") with no
      commandContext() reference.

- 1 Consumer: none — this baseline has no Consumer beyond the workflow start path.
  MessageQueue's InboundMessageReceived events trigger TriageWorkflow directly via
  MessageQueue.receive → workflow start inside TriageEndpoint.

- 1 View MessageView with row type MessageRow (mirrors Message; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes MessageEntity events.
  ONE query getAllMessages SELECT * AS messages FROM message_view. No WHERE category or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction MessageSimulator — every 30s, reads next line from
  src/main/resources/sample-events/support-messages.jsonl (loops at EOF) and calls
  MessageQueue.receive with a fresh messageId (UUID), then starts a TriageWorkflow
  with the same messageId.

- 2 HttpEndpoints:
    * TriageEndpoint at /api with GET /messages (list from MessageView.getAllMessages,
      filter client-side by ?category and ?status query params), GET /messages/{id},
      POST /messages (body SupportMessage minus messageId/receivedAt — server assigns),
      POST /messages/{id}/unblock (body {decidedBy, note} — operator override:
      publishes the blocked draft as RESOLVED with an audit note),
      GET /messages/sse (serverSentEventsForView over getAllMessages), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TriageTasks.java declaring the task constants: RESOLVE (resultConformsTo HandlerReply.class,
  description "Resolve the support message end-to-end and return a typed HandlerReply").
- Domain records SupportMessage, RouteDecision, HandlerReply, GuardrailResult, and the
  Message entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9573 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/support-messages.jsonl with 9 canned lines (3 BILLING-
  flavoured, 3 PRODUCT-flavoured, 2 UNCLEAR-flavoured, 1 designed to trip the guardrail
  with a specific-timeline refund promise).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control: G1 guardrail before-agent-response
  policy-rubric. Matching simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  blocked drafts wait for a human), failure.failure_modes including "wrong-category-routing",
  "inappropriate-reply-content", "refund-amount-fabrication"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router-agent.md, prompts/billing-handler.md, prompts/product-handler.md,
  prompts/draft-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Triage Agent (UX)", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = message list with category chip + status pill; centre = message body + route
  decision; right = handler draft + guardrail verdict + published response or violations
  + Unblock button). Browser title exactly:
  <title>Akka Sample: Triage Agent (UX)</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by messageId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    router-agent.json — 10 RouteDecision entries spanning BILLING (charge
      disputes, refund requests, plan changes), PRODUCT (API errors,
      usage questions, integration help, performance reports), and UNCLEAR
      (ambiguous, off-topic, single-word). Confidence + a one-sentence
      reason on each.
    billing-handler.json — 7 HandlerReply entries: 4 with action
      INFO_PROVIDED or REFUND_INITIATED (within authority), 1 with action
      ESCALATED, 1 with ARTICLE_LINKED, 1 designed to trip the guardrail
      (promises a "next-business-day refund" — outside published SLA).
    product-handler.json — 7 HandlerReply entries: 4 with INFO_PROVIDED
      or ARTICLE_LINKED, 1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED,
      1 designed to trip the guardrail (gives specific regulatory-compliance
      advice outside handler scope).
    draft-guardrail.json — 8 GuardrailResult entries. 5 with allowed=true
      and empty violations. 3 with allowed=false and one violation each
      ("invented-refund-timeline", "out-of-scope-compliance-advice",
      "echoes-redacted-token"). The mock matches the trip-the-guardrail
      entries above when called for the same messageId.
- A MockModelProvider.seedFor(messageId) helper makes per-message selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  BillingHandler and ProductHandler both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): routeStep 20s,
  guardrailStep 20s, billingStep / productStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Message is Optional<T>. The
  MessageView row type uses the same Optional wrapping.
- (Lesson 7) TriageTasks.java declares the RESOLVE Task<HandlerReply> constant.
  Both handlers' definition().capability(TaskAcceptance.of(RESOLVE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9573 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The guardrail step happens BEFORE ReplyPublished. A blocked draft
  never reaches the UI as published — only as a "blocked draft + violations"
  surface for the operator.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names.
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
