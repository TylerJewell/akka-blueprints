# SPEC — triage-handoff-routing

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SK Multi-Agent Handoff.
**One-line pitch:** A triage agent classifies an incoming conversation turn by intent and hands the resolution off to an account, product, or returns specialist that owns it end-to-end, with a routing-decision guardrail before specialist invocation and a response-content guardrail before the reply reaches the user.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the task, then transfers the same task identity to a downstream specialist agent that produces the final output. Two guardrail mechanisms are layered on top:

- A **before-agent-invocation guardrail** runs after the triage decision but before the chosen specialist is invoked. It validates that the routing decision meets a minimum confidence threshold and that the chosen specialist is within its declared scope. A routing decision that fails this check never reaches the specialist — the turn enters `ROUTING_BLOCKED` for human review.
- A **before-agent-response guardrail** runs on the specialist's draft reply. It checks the draft against a content policy (no invented policy citations, no account-number echoes, no out-of-scope financial advice, no promises outside the published return window). A draft that fails this check enters `RESPONSE_BLOCKED` for human review.

The pattern is a textbook fan-out-of-one: the workflow branches on the classifier's intent category, and only the chosen specialist is invoked. The other specialists see no traffic for that turn.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live conversation list. Every turn displays its intent chip, status pill, routing verdict chip, and (if resolved) the published reply.
2. `ConversationSimulator` (TimedAction) ticks every 30 s and inserts a new canned turn from `sample-events/conversation-turns.jsonl` into `ConversationQueue`.
3. For each new turn: `IntentClassifier` (Consumer) normalises the raw text, registers a `ConversationEntity`, and starts a `HandoffWorkflow`.
4. The workflow calls `TriageAgent`, gets a `TriageDecision { intent, confidence, reason }`, and emits `TurnTriaged` on the entity.
5. The workflow calls `RoutingGuardrail` with the decision to check it before invoking the specialist. If the verdict is `BLOCKED`, the workflow emits `RoutingBlocked` (terminal `ROUTING_BLOCKED`). If `ALLOWED`, it proceeds.
6. Branch on `intent`:
   - `ACCOUNT` → workflow calls `AccountSpecialist` with the `RESOLVE` task.
   - `PRODUCT` → workflow calls `ProductSpecialist` with the `RESOLVE` task.
   - `RETURNS` → workflow calls `ReturnSpecialist` with the `RESOLVE` task.
   - `UNCLEAR` → workflow emits `TurnEscalated`; ends.
7. The specialist's draft `Reply` passes through the `ResponseGuardrail`. If accepted, `ReplyPublished` is emitted (terminal `RESOLVED`). If rejected, `ReplyBlocked` is emitted (terminal `RESPONSE_BLOCKED`) with the violation list.
8. The user can click any turn card and see the normalised input, the triage decision, the routing verdict, the chosen specialist, and the draft (or blocked draft + violations + Review button).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationSimulator` | `TimedAction` | Drips simulated conversation turns into `ConversationQueue` every 30 s. | scheduler | `ConversationQueue` |
| `ConversationQueue` | `EventSourcedEntity` | Append-only audit log of every inbound turn (`InboundTurnReceived`). | `ConversationSimulator`, `HandoffEndpoint` | `IntentClassifier` |
| `IntentClassifier` | `Consumer` | Subscribes to `ConversationQueue` events; normalises the raw turn; registers `ConversationEntity`; starts a `HandoffWorkflow`. | `ConversationQueue` events | `ConversationEntity`, `HandoffWorkflow` |
| `TriageAgent` | `Agent` (typed, not autonomous) | Classifies a `NormalisedTurn` into `ACCOUNT` / `PRODUCT` / `RETURNS` / `UNCLEAR` with confidence + reason. | invoked by `HandoffWorkflow` | returns `TriageDecision` |
| `RoutingGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: validates the routing decision before the specialist is invoked. Returns `RoutingVerdict { allowed, reason, rubricVersion }`. | invoked by `HandoffWorkflow` | returns `RoutingVerdict` |
| `AccountSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for account-intent turns. Returns typed `Reply`. | invoked by `HandoffWorkflow` | returns `Reply` |
| `ProductSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for product-intent turns. Returns typed `Reply`. | invoked by `HandoffWorkflow` | returns `Reply` |
| `ReturnSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for returns-intent turns. Returns typed `Reply`. | invoked by `HandoffWorkflow` | returns `Reply` |
| `ResponseGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks the specialist's draft against the content policy. Returns `ResponseVerdict { allowed, violations, rubricVersion }`. | invoked by `HandoffWorkflow` | returns `ResponseVerdict` |
| `HandoffWorkflow` | `Workflow` | Per-turn orchestration: triage → routing-check → branch → specialist → response-check → publish. | `IntentClassifier` (start) | `ConversationEntity` |
| `ConversationEntity` | `EventSourcedEntity` | Per-turn lifecycle. | `HandoffWorkflow`, `IntentClassifier` | `ConversationView` |
| `ConversationView` | `View` | Read-model row per turn. | `ConversationEntity` events | `HandoffEndpoint` |
| `HandoffEndpoint` | `HttpEndpoint` | `/api/turns/*` — list, get, manual submit, manual review, SSE; `/api/metadata/*`. | — | `ConversationView`, `ConversationEntity`, `ConversationQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundTurn(
    String turnId,
    String sessionId,
    String channel,            // "chat" | "email" | "web-widget"
    String rawText,
    Instant receivedAt
) {}

record NormalisedTurn(
    String normalisedText,
    String languageCode,       // "en" | "fr" | "de" | ...
    boolean containsSensitiveData  // true if the classifier flagged PII-adjacent content
) {}

enum IntentCategory { ACCOUNT, PRODUCT, RETURNS, UNCLEAR }

record TriageDecision(
    IntentCategory intent,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

record RoutingVerdict(
    boolean allowed,
    String reason,             // one short sentence — what passed or failed
    String rubricVersion       // "v1"
) {}

enum ReplyAction { INFORMATION_PROVIDED, ESCALATED, ACCOUNT_UPDATED, RETURN_INITIATED, FOLLOW_UP_SCHEDULED }

record Reply(
    String responseText,
    ReplyAction action,
    String specialistTag,      // "account" | "product" | "returns"
    Instant repliedAt
) {}

record ResponseVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion       // "v1"
) {}

record Conversation(
    String turnId,
    InboundTurn inbound,
    Optional<NormalisedTurn> normalised,
    Optional<TriageDecision> triage,
    Optional<RoutingVerdict> routing,
    Optional<Reply> reply,
    Optional<ResponseVerdict> responseVerdict,
    Optional<String> escalationReason,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ConversationStatus {
    RECEIVED,
    NORMALISED,
    TRIAGED,
    ROUTING_BLOCKED,
    ROUTED_ACCOUNT,
    ROUTED_PRODUCT,
    ROUTED_RETURNS,
    REPLY_DRAFTED,
    RESPONSE_BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `ConversationEntity`: `TurnRegistered`, `TurnNormalised`, `TurnTriaged`, `RoutingBlocked`, `TurnRouted`, `ReplyDrafted`, `ResponseVerdictAttached`, `ReplyPublished`, `ReplyBlocked`, `TurnEscalated`.

Events on `ConversationQueue`: `InboundTurnReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/turns` — list all turns (newest-first), optional `?intent=ACCOUNT|PRODUCT|RETURNS|UNCLEAR&status=…` filtered client-side.
- `GET /api/turns/{id}` — one turn.
- `POST /api/turns` — manually submit a turn (body `InboundTurn` minus `turnId` and `receivedAt`); server assigns both.
- `POST /api/turns/{id}/review` — body `{ reviewedBy, note, publish }` — operator override; if `publish=true` transitions `ROUTING_BLOCKED` or `RESPONSE_BLOCKED` to `RESOLVED`; if `publish=false` transitions to `ESCALATED`.
- `GET /api/turns/sse` — Server-Sent Events for every turn change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SK Multi-Agent Handoff</title>`.

The App UI tab is a three-pane layout: **left** is the turn list (status pill + intent chip + routing verdict chip), **centre** is the selected turn's normalised text + triage decision + routing verdict, **right** is the chosen specialist's draft + response verdict + published reply (or violations + Review button when blocked).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail** on the routing decision: `RoutingGuardrail` checks that the triage confidence meets a minimum threshold and that the chosen specialist is within its declared intent scope. A low-confidence or out-of-scope routing decision is blocked before the specialist sees the turn. Blocking — a failed routing check transitions to `ROUTING_BLOCKED` for human review.
- **G2 — before-agent-response guardrail** on every specialist's draft: `ResponseGuardrail` checks the draft against a content policy (no invented policy citations, no account-number echoes, no out-of-scope financial advice, no return-window promises outside published policy). Blocking — a violation transitions to `RESPONSE_BLOCKED` for human review.

## 9. Agent prompts

- `TriageAgent` → `prompts/triage-agent.md`. Typed classifier; returns one of `ACCOUNT`, `PRODUCT`, `RETURNS`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `RoutingGuardrail` → `prompts/routing-guardrail.md`. Before-agent-invocation guardrail; returns a `RoutingVerdict { allowed, reason, rubricVersion }`. Conservative — borderline routing decisions are blocked.
- `AccountSpecialist` → `prompts/account-specialist.md`. Owns the `RESOLVE` task for account-intent turns.
- `ProductSpecialist` → `prompts/product-specialist.md`. Owns the `RESOLVE` task for product-intent turns.
- `ReturnSpecialist` → `prompts/return-specialist.md`. Owns the `RESOLVE` task for returns-intent turns.
- `ResponseGuardrail` → `prompts/response-guardrail.md`. Returns a `ResponseVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an account-intent turn → routing guardrail passes → `AccountSpecialist` resolves → response guardrail passes → published.
2. **J2** — Simulator drips a product-intent turn → routing guardrail passes → `ProductSpecialist` resolves → response guardrail passes → published.
3. **J3** — Simulator drips a returns-intent turn → routing guardrail passes → `ReturnSpecialist` resolves → response guardrail passes → published.
4. **J4** — A turn with low-confidence triage output fails the routing guardrail; lands in `ROUTING_BLOCKED`; operator can override via Review.
5. **J5** — A specialist draft that violates the response policy is blocked; lands in `RESPONSE_BLOCKED`; operator can override via Review.
6. **J6** — An ambiguous turn triages as `UNCLEAR` and terminates in `ESCALATED`; no specialist is invoked.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named triage-handoff-routing demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real chat integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-triage-handoff-routing.
Java package io.akka.samples.skmultiagenthandoff. Akka 3.6.0. HTTP port 9774.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) TriageAgent — classifier. System prompt loaded from
  prompts/triage-agent.md. Input: NormalisedTurn{normalisedText, languageCode,
  containsSensitiveData: boolean}. Output: TriageDecision{intent: IntentCategory
  (ACCOUNT/PRODUCT/RETURNS/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 Agent (typed) RoutingGuardrail — before-agent-invocation guardrail. System prompt from
  prompts/routing-guardrail.md. Input: NormalisedTurn + TriageDecision. Output:
  RoutingVerdict{allowed: boolean, reason: String, rubricVersion: String = "v1"}.
  Used by HandoffWorkflow.routingCheckStep before invoking any specialist; blocking.

- 1 AutonomousAgent AccountSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/account-specialist.md. Input:
  NormalisedTurn + TriageDecision. Output: Reply{responseText, action: ReplyAction,
  specialistTag = "account", repliedAt}. Never invents policy citations; sets
  action=ESCALATED when outside authority.

- 1 AutonomousAgent ProductSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/product-specialist.md. Same input
  shape; specialistTag = "product".

- 1 AutonomousAgent ReturnSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/return-specialist.md. Same input
  shape; specialistTag = "returns".

- 1 Agent (typed) ResponseGuardrail — before-agent-response guardrail. System prompt from
  prompts/response-guardrail.md. Input: NormalisedTurn + Reply. Output:
  ResponseVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by HandoffWorkflow.responseCheckStep before publishing; blocking.

- 1 Workflow HandoffWorkflow per turnId. Steps:
    triageStep -> routingCheckStep -> {accountStep | productStep | returnStep | escalateStep}
                -> responseCheckStep -> publishStep
  triageStep calls componentClient.forAgent().inSession(turnId).method(TriageAgent::triage)
    .invoke(normalised). On success emits TurnTriaged via ConversationEntity.recordTriage.
  routingCheckStep calls forAgent(...).method(RoutingGuardrail::check).invoke(normalised, decision).
    On verdict.allowed=false emit RoutingBlocked (terminal ROUTING_BLOCKED) and end.
    On verdict.allowed=true proceed to branch.
  Branch on TriageDecision.intent:
    ACCOUNT -> proceed to accountStep (emits TurnRouted{ACCOUNT})
    PRODUCT -> proceed to productStep (emits TurnRouted{PRODUCT})
    RETURNS -> proceed to returnStep (emits TurnRouted{RETURNS})
    UNCLEAR -> escalateStep (emits TurnEscalated; terminates).
  accountStep / productStep / returnStep call forAutonomousAgent(<Specialist>.class, turnId)
    .runSingleTask(TaskDef.instructions(buildPrompt(normalised, decision))) returning a taskId,
    then forTask(taskId).result(HandoffTasks.RESOLVE) to block on the typed Reply.
    On success emits ReplyDrafted.
  responseCheckStep calls forAgent(...).method(ResponseGuardrail::check).invoke(normalised, draft).
    On verdict.allowed=true proceed to publishStep (emits ReplyPublished, terminal RESOLVED).
    On verdict.allowed=false emit ReplyBlocked (terminal RESPONSE_BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on triageStep and
    routingCheckStep and responseCheckStep, stepTimeout(Duration.ofSeconds(60)) on
    accountStep, productStep, returnStep, and publishStep.
    defaultStepRecovery(maxRetries(2).failoverTo(HandoffWorkflow::error)).

- 2 EventSourcedEntities:
    * ConversationQueue — append-only audit log. Command receive(InboundTurn) emits
      InboundTurnReceived{inbound}. No mutable state beyond a counter; commands are
      idempotent on inbound.turnId.
    * ConversationEntity (one per turnId) — full per-turn lifecycle. State
      Conversation{turnId, inbound: InboundTurn, Optional<NormalisedTurn> normalised,
      Optional<TriageDecision> triage, Optional<RoutingVerdict> routing,
      Optional<Reply> reply, Optional<ResponseVerdict> responseVerdict,
      Optional<String> escalationReason, ConversationStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. ConversationStatus enum: RECEIVED, NORMALISED,
      TRIAGED, ROUTING_BLOCKED, ROUTED_ACCOUNT, ROUTED_PRODUCT, ROUTED_RETURNS,
      REPLY_DRAFTED, RESPONSE_BLOCKED, RESOLVED, ESCALATED.
      Events: TurnRegistered, TurnNormalised, TurnTriaged, RoutingBlocked, TurnRouted,
      ReplyDrafted, ResponseVerdictAttached, ReplyPublished, ReplyBlocked, TurnEscalated.
      Commands: registerTurn, attachNormalised, recordTriage, recordRoutingVerdict,
      recordRouting, recordDraft, recordResponseVerdict, publish, blockResponse,
      blockRouting, escalate, review, getTurn. emptyState() returns
      Conversation.initial("") with no commandContext() reference.

- 1 Consumer:
    * IntentClassifier subscribed to ConversationQueue events; for each InboundTurnReceived
      applies a normalisation pass (lower-case, whitespace collapse, language detection via
      a simple heuristic; flags containsSensitiveData if the text matches patterns for
      credit-card-adjacent, account-number-adjacent, or national-id-adjacent tokens) to
      rawText, builds NormalisedTurn, and calls ConversationEntity.registerTurn then
      attachNormalised for the turnId; then starts a HandoffWorkflow with turnId as the
      workflow id.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation; uses
  Optional<T> for every nullable lifecycle field per Lesson 6). Table updater consumes
  ConversationEntity events. ONE query getAllTurns SELECT * AS turns FROM conversation_view.
  No WHERE intent or WHERE status filter (Akka cannot auto-index enum columns) — filter
  client-side in callers.

- 1 TimedAction ConversationSimulator — every 30s, reads next line from
  src/main/resources/sample-events/conversation-turns.jsonl (loops at EOF) and calls
  ConversationQueue.receive with a fresh turnId (UUID).

- 2 HttpEndpoints:
    * HandoffEndpoint at /api with GET /turns (list from ConversationView.getAllTurns,
      filter client-side by ?intent and ?status query params), GET /turns/{id},
      POST /turns (body InboundTurn minus turnId/receivedAt — server assigns),
      POST /turns/{id}/review (body {reviewedBy, note, publish: boolean} — operator override:
      if publish=true transitions ROUTING_BLOCKED or RESPONSE_BLOCKED to RESOLVED with an
      audit note; if publish=false transitions to ESCALATED),
      GET /turns/sse (serverSentEventsForView over getAllTurns), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- HandoffTasks.java declaring the task constants: RESOLVE (resultConformsTo Reply.class,
  description "Resolve the conversation turn end-to-end and return a typed Reply").
- Domain records InboundTurn, NormalisedTurn, TriageDecision, RoutingVerdict, Reply,
  ResponseVerdict, and the Conversation entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9774 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/conversation-turns.jsonl with 9 canned lines (3 ACCOUNT-
  intent, 3 PRODUCT-intent, 1 RETURNS-intent, 1 UNCLEAR-intent, 1 designed to trip the
  routing guardrail via a low-confidence ambiguous mixed-intent turn, and 1 designed to trip
  the response guardrail with an invented policy citation).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 routing guardrail
  before-agent-invocation, G2 response guardrail before-agent-response. Matching
  simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  decisions.authority_level = draft-only, oversight.human_in_loop = false (the system
  publishes without HITL by default — only blocked turns wait for a human); deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/triage-agent.md, prompts/routing-guardrail.md, prompts/account-specialist.md,
  prompts/product-specialist.md, prompts/return-specialist.md, prompts/response-guardrail.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: SK Multi-Agent Handoff", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = turn list with intent chip + status pill + routing verdict chip; centre = normalised
  text + triage block + routing verdict block; right = specialist draft + response verdict +
  published reply or violations + Review button). Browser title exactly:
  <title>Akka Sample: SK Multi-Agent Handoff</title>. No subtitle on the Overview tab.

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
  pseudo-randomly per call (seeded by turnId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    triage-agent.json — 12 TriageDecision entries spanning ACCOUNT (login
      issues, subscription management, password reset), PRODUCT (feature
      questions, integration how-tos, capability questions), RETURNS (refund
      status, exchange requests, return label requests), and UNCLEAR (mixed
      content, very short messages, off-topic). Confidence + a one-sentence
      reason on each.
    routing-guardrail.json — 10 RoutingVerdict entries. 8 with allowed=true.
      2 with allowed=false: one for a low-confidence routing to ACCOUNT where
      the turn is clearly about RETURNS (reason "confidence low and intent
      mismatch"), one for a turn whose text contains only punctuation
      (reason "normalised text too short to validate routing").
    account-specialist.json — 8 Reply entries: 5 with action
      INFORMATION_PROVIDED or ACCOUNT_UPDATED (well within authority), 1
      with action ESCALATED (account closure outside authority), 1 with
      action FOLLOW_UP_SCHEDULED, 1 designed to trip the response guardrail
      (cites a made-up policy number "POL-2091") so guardrail tests have
      material.
    product-specialist.json — 8 Reply entries: 5 with INFORMATION_PROVIDED,
      1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED, 1 designed to trip the
      response guardrail (gives out-of-scope financial advice about pricing
      discounts outside authority).
    return-specialist.json — 8 Reply entries: 5 with INFORMATION_PROVIDED or
      RETURN_INITIATED, 1 with ESCALATED, 1 with FOLLOW_UP_SCHEDULED, 1
      designed to trip the response guardrail (promises a return window
      outside the published 30-day policy).
    response-guardrail.json — 10 ResponseVerdict entries. 7 with
      allowed=true and empty violations. 3 with allowed=false and one
      violation each ("invented-policy-citation", "out-of-scope-financial-advice",
      "return-window-outside-policy"). The mock should match the
      guardrail-tripping entries above when called for the same turnId.
- A MockModelProvider.seedFor(turnId) helper makes per-turn selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  AccountSpecialist, ProductSpecialist, and ReturnSpecialist all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): triageStep 20s,
  routingCheckStep 20s, responseCheckStep 20s, accountStep / productStep /
  returnStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Conversation is Optional<T>. The
  ConversationView row type uses the same Optional wrapping.
- (Lesson 7) HandoffTasks.java declares the RESOLVE Task<Reply> constant.
  All three specialists' definition().capability(TaskAcceptance.of(RESOLVE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9774 in application.conf; not 9000.
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
- The IntentClassifier runs INSIDE a Consumer before any LLM call — the
  normalised text is what reaches the agents; agents never see the raw turn.
- The RoutingGuardrail step happens BEFORE the specialist is invoked. A
  blocked routing decision never causes a specialist to be called.
- The ResponseGuardrail step happens BEFORE ReplyPublished. A blocked draft
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
