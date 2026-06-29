# SPEC — akka-handoff-routing

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** OpenAI Agents SDK Handoff Routing.
**One-line pitch:** A triage agent classifies an inbound conversation turn and hands it off to a billing or technical specialist with full conversation context — demonstrating the OpenAI Agents SDK handoff primitive running durably in Akka, with a before-handoff routing guardrail and PII filtering on the message context bundle.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the conversation, then transfers full conversation context to a downstream specialist that produces the final reply. This blueprint specifically models the OpenAI Agents SDK `handoff()` primitive and its message-filter variant: the context bundle passed to the specialist is filtered to remove raw PII before the handoff executes.

Two governance mechanisms are layered on top:

- A **PII sanitizer** (`MessageFilter` Consumer) runs between the raw conversation event and the LLM call. The classifier and specialists never see raw emails, phone numbers, card numbers, or account ids.
- A **before-agent-invocation guardrail** (`RoutingGuardrail` Agent) runs on the routing context *before* the specialist is invoked. It checks whether the context bundle is safe to hand off — no unredacted identifiers, no out-of-scope content, no prompt-injection signals. If it rejects, the conversation is blocked before the specialist is ever called.
- An **on-decision eval** fires every time the triage agent emits a routing decision. A `HandoffJudge` agent grades the decision against the filtered payload on a 1–5 rubric. The score and rationale are written back to the conversation and surfaced in the UI.

The message-filter variant is the distinguishing feature: unlike a plain handoff, the context bundle is processed by `MessageFilter` before it leaves the originating agent's boundary. This is the mechanism the OpenAI Agents SDK calls `input_filter` on a handoff definition.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live conversation list. Every row displays its routing chip, status pill, handoff score, and (if resolved) the published reply.
2. `ConversationSimulator` (TimedAction) ticks every 30 s and inserts a new canned turn from `sample-events/conversations.jsonl` into `ConversationQueue`.
3. For each new turn: `MessageFilter` (Consumer) redacts the payload, registers a `ConversationEntity`, and starts a `HandoffWorkflow`.
4. The workflow calls `RoutingGuardrail` with the filtered context. If the context passes, the workflow calls `TriageAgent` to get a `RoutingDecision { category, confidence, reason }` and emits `RoutingDecided` on the entity.
5. Branch on `category`:
   - `BILLING` → workflow calls `BillingSpecialist` with the `RESOLVE` task and the filtered context bundle.
   - `TECHNICAL` → workflow calls `TechnicalSpecialist` with the same task and context.
   - `UNCLEAR` → workflow emits `ConversationEscalated`; ends.
6. The specialist returns a typed `Reply`. The workflow emits `ReplyPublished` (terminal `RESOLVED`).
7. If `RoutingGuardrail` rejects, the workflow emits `HandoffBlocked` (terminal `BLOCKED`). The operator can review via `POST /api/conversations/{id}/unblock`.
8. Independent of the workflow, `HandoffEvalScorer` (Consumer) listens for `RoutingDecided` events, calls `HandoffJudge`, and writes `HandoffScored { score, rationale }` back to the conversation entity.
9. The user can click any conversation row and see the filtered context, the routing decision, the handoff score, the chosen specialist, and the published reply (or the blocked context + guardrail violations).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationSimulator` | `TimedAction` | Drips simulated conversation turns into `ConversationQueue` every 30 s. | scheduler | `ConversationQueue` |
| `ConversationQueue` | `EventSourcedEntity` | Append-only audit log of every inbound turn (`InboundTurnReceived`). | `ConversationSimulator`, `HandoffEndpoint` | `MessageFilter` |
| `MessageFilter` | `Consumer` | Subscribes to `ConversationQueue` events; redacts PII; registers `ConversationEntity`; starts a `HandoffWorkflow`. | `ConversationQueue` events | `ConversationEntity`, `HandoffWorkflow` |
| `RoutingGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: checks the filtered context bundle for safety before any specialist is invoked. Returns `GuardrailVerdict { allowed, violations, rubricVersion }`. | invoked by `HandoffWorkflow` | returns `GuardrailVerdict` |
| `TriageAgent` | `Agent` (typed, not autonomous) | Classifies a `FilteredContext` into `BILLING` / `TECHNICAL` / `UNCLEAR` with confidence + reason. | invoked by `HandoffWorkflow` | returns `RoutingDecision` |
| `BillingSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for billing conversations. Returns typed `Reply`. | invoked by `HandoffWorkflow` | returns `Reply` |
| `TechnicalSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for technical conversations. Returns typed `Reply`. | invoked by `HandoffWorkflow` | returns `Reply` |
| `HandoffJudge` | `Agent` (typed) | Grades a routing decision against the filtered context. Returns `HandoffScore { score 1–5, rationale }`. | invoked by `HandoffEvalScorer` | returns `HandoffScore` |
| `HandoffWorkflow` | `Workflow` | Per-conversation orchestration: filter-confirm → guardrail → triage → route → resolve → publish. | `MessageFilter` (start) | `ConversationEntity` |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle. | `HandoffWorkflow`, `HandoffEvalScorer` | `ConversationView` |
| `ConversationView` | `View` | Read-model row per conversation. | `ConversationEntity` events | `HandoffEndpoint` |
| `HandoffEvalScorer` | `Consumer` | Subscribes to `ConversationEntity` events; on `RoutingDecided` invokes `HandoffJudge` and writes `HandoffScored` back. | `ConversationEntity` events | `ConversationEntity` |
| `HandoffEndpoint` | `HttpEndpoint` | `/api/conversations/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `ConversationView`, `ConversationEntity`, `ConversationQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundTurn(
    String conversationId,
    String customerId,
    String channel,           // "chat" | "email" | "web-form"
    String userMessage,
    List<String> priorTurns,  // filtered summaries of earlier turns
    Instant receivedAt
) {}

record FilteredContext(
    String filteredMessage,
    List<String> filteredPriorTurns,
    List<String> piiCategoriesFound
) {}

enum RoutingCategory { BILLING, TECHNICAL, UNCLEAR }

record RoutingDecision(
    RoutingCategory category,
    String confidence,        // "high" | "medium" | "low"
    String reason             // one short sentence
) {}

enum ReplyAction { REFUND_INITIATED, ARTICLE_LINKED, FOLLOW_UP_SCHEDULED, INFO_PROVIDED, ESCALATED }

record Reply(
    String replyBody,
    ReplyAction action,
    String specialistTag,     // "billing" | "technical"
    Instant repliedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,  // empty when allowed
    String rubricVersion
) {}

record HandoffScore(
    int score,                // 1..5
    String rationale,
    Instant scoredAt
) {}

record Conversation(
    String conversationId,
    InboundTurn inbound,
    Optional<FilteredContext> filtered,
    Optional<GuardrailVerdict> guardrail,
    Optional<RoutingDecision> routing,
    Optional<Reply> reply,
    Optional<HandoffScore> handoffScore,
    Optional<String> escalationReason,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ConversationStatus {
    RECEIVED,
    FILTERED,
    GUARDRAIL_PASSED,
    ROUTED_BILLING,
    ROUTED_TECHNICAL,
    REPLY_READY,
    BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `ConversationEntity`: `ConversationRegistered`, `ConversationFiltered`, `GuardrailVerdictAttached`, `RoutingDecided`, `ConversationRouted`, `ReplyDrafted`, `ReplyPublished`, `HandoffBlocked`, `ConversationEscalated`, `HandoffScored`.

Events on `ConversationQueue`: `InboundTurnReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/conversations` — list all conversations (newest-first), optional `?category=BILLING|TECHNICAL|UNCLEAR&status=…` filtered client-side.
- `GET /api/conversations/{id}` — one conversation.
- `POST /api/conversations` — manually submit a turn (body `InboundTurn` minus `conversationId` and `receivedAt`); server assigns both.
- `POST /api/conversations/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator chooses to release the conversation.
- `GET /api/conversations/sse` — Server-Sent Events for every conversation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: OpenAI Agents SDK Handoff Routing</title>`.

The App UI tab is a three-pane layout: **left** is the conversation list (status pill + routing chip + score chip), **centre** is the selected conversation's filtered context + routing decision + score, **right** is the chosen specialist's reply + guardrail verdict + published reply (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `MessageFilter` Consumer): redacts emails, phone numbers, card numbers, and account ids from the inbound message and prior-turn summaries before any LLM sees them. The categories found are kept for audit; the filtered text is what reaches the agents.
- **G1 — before-agent-invocation guardrail** on the handoff boundary (before `BillingSpecialist` or `TechnicalSpecialist` is invoked): `RoutingGuardrail` checks the filtered context bundle for unredacted identifiers, prompt-injection signals, and out-of-scope content. Blocking — a violation puts the conversation in `BLOCKED` for human review.
- **E1 — on-decision eval** (`eval-event`, on the routing decision): `HandoffEvalScorer` (Consumer) listens for `RoutingDecided` events and calls `HandoffJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate.

## 9. Agent prompts

- `TriageAgent` → `prompts/triage-agent.md`. Typed classifier; returns one of `BILLING`, `TECHNICAL`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `BillingSpecialist` → `prompts/billing-specialist.md`. Owns the `RESOLVE` task for billing conversations. Never invents refund amounts; escalates outside its authority.
- `TechnicalSpecialist` → `prompts/technical-specialist.md`. Owns the `RESOLVE` task for technical conversations. Cites a help-centre article id when one applies; never invents one.
- `HandoffJudge` → `prompts/handoff-judge.md`. Grades a routing decision against a 1–5 rubric with one-sentence rationale.
- `RoutingGuardrail` → `prompts/routing-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline contexts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a billing-flavoured turn → guardrail passes → triaged `BILLING` → resolved by `BillingSpecialist` → published.
2. **J2** — Simulator drips a technical turn → guardrail passes → triaged `TECHNICAL` → resolved by `TechnicalSpecialist` → published.
3. **J3** — An ambiguous turn triages as `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
4. **J4** — A context bundle containing a prompt-injection signal is blocked by the routing guardrail; the conversation lands in `BLOCKED` with the violation listed; the operator can unblock.
5. **J5** — Every routed conversation carries a `HandoffScore` (1–5) and rationale within ~10 s of the routing decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-handoff-routing demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real messaging integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-akka-handoff-routing.
Java package io.akka.samples.openaiagentssdkhandoffrouting. Akka 3.6.0. HTTP port 9438.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) TriageAgent — classifier. System prompt loaded from
  prompts/triage-agent.md. Input: FilteredContext{filteredMessage, filteredPriorTurns:
  List<String>, piiCategoriesFound: List<String>}. Output: RoutingDecision{category:
  RoutingCategory (BILLING/TECHNICAL/UNCLEAR), confidence: "high"|"medium"|"low",
  reason: String}. Defaults to UNCLEAR under uncertainty.

- 1 Agent (typed) RoutingGuardrail — before-agent-invocation guardrail. System prompt from
  prompts/routing-guardrail.md. Input: FilteredContext + RoutingDecision (if available;
  null on first check). Output: GuardrailVerdict{allowed: boolean, violations: List<String>,
  rubricVersion: String}. Used by HandoffWorkflow before invoking any specialist; blocking.

- 1 AutonomousAgent BillingSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/billing-specialist.md. Input:
  FilteredContext + RoutingDecision. Output: Reply{replyBody, action: ReplyAction,
  specialistTag = "billing", repliedAt}. Never invents refund amounts; sets action=ESCALATED
  with a reason when outside authority.

- 1 AutonomousAgent TechnicalSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/technical-specialist.md. Same
  input shape; specialistTag = "technical".

- 1 Agent (typed) HandoffJudge — judge. System prompt from prompts/handoff-judge.md. Input:
  FilteredContext + RoutingDecision. Output: HandoffScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Workflow HandoffWorkflow per conversationId. Steps:
    filterConfirmStep -> guardrailStep -> {triageStep} -> routeStep ->
      {billingStep | technicalStep | escalateStep} -> publishStep
  filterConfirmStep verifies FilteredContext is attached (emits ConversationFiltered);
    proceeds immediately to guardrailStep.
  guardrailStep calls componentClient.forAgent().inSession(conversationId)
    .method(RoutingGuardrail::check).invoke(filtered, null). On allowed=true emits
    GuardrailVerdictAttached (allowed) and proceeds to triageStep. On allowed=false
    emits GuardrailVerdictAttached (rejected) then HandoffBlocked (terminal BLOCKED); ends.
  triageStep calls componentClient.forAgent().inSession(conversationId)
    .method(TriageAgent::classify).invoke(filtered). On success emits RoutingDecided via
    ConversationEntity.recordRouting.
  routeStep branches on RoutingDecision.category:
    BILLING -> proceed to billingStep (emits ConversationRouted{BILLING})
    TECHNICAL -> proceed to technicalStep (emits ConversationRouted{TECHNICAL})
    UNCLEAR -> escalateStep (emits ConversationEscalated; terminates).
  billingStep / technicalStep call forAutonomousAgent(<Specialist>.class, conversationId)
    .runSingleTask(TaskDef.instructions(buildPrompt(filtered, routing))) returning a taskId,
    then forTask(taskId).result(HandoffTasks.RESOLVE) to block on the typed Reply.
    On success emits ReplyDrafted.
  publishStep emits ReplyPublished (terminal RESOLVED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on filterConfirmStep,
    guardrailStep, and triageStep; stepTimeout(Duration.ofSeconds(60)) on billingStep,
    technicalStep, and publishStep. defaultStepRecovery(maxRetries(2)
    .failoverTo(HandoffWorkflow::error)).

- 2 EventSourcedEntities:
    * ConversationQueue — append-only audit log. Command receive(InboundTurn) emits
      InboundTurnReceived{inbound}. No mutable state beyond a counter; commands are
      idempotent on inbound.conversationId.
    * ConversationEntity (one per conversationId) — full per-conversation lifecycle.
      State Conversation{conversationId, inbound: InboundTurn, Optional<FilteredContext>
      filtered, Optional<GuardrailVerdict> guardrail, Optional<RoutingDecision> routing,
      Optional<Reply> reply, Optional<HandoffScore> handoffScore,
      Optional<String> escalationReason, ConversationStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. ConversationStatus enum: RECEIVED, FILTERED,
      GUARDRAIL_PASSED, ROUTED_BILLING, ROUTED_TECHNICAL, REPLY_READY, BLOCKED,
      RESOLVED, ESCALATED. Events: ConversationRegistered, ConversationFiltered,
      GuardrailVerdictAttached, RoutingDecided, ConversationRouted, ReplyDrafted,
      ReplyPublished, HandoffBlocked, ConversationEscalated, HandoffScored. Commands:
      registerInbound, attachFiltered, attachGuardrailVerdict, recordRouting, recordRouted,
      recordDraft, publish, block, escalate, recordHandoffScore, unblock, getConversation.
      emptyState() returns Conversation.initial("") with no commandContext() reference.

- 2 Consumers:
    * MessageFilter subscribed to ConversationQueue events; for each InboundTurnReceived
      applies a regex+heuristic redaction pipeline (emails, phone numbers, payment-card
      numbers via Luhn-aware regex, account ids matching ACCT-\w+, plus heuristic name-
      redaction in greeting positions) to userMessage and each element of priorTurns,
      builds FilteredContext with piiCategoriesFound, calls ConversationEntity
      .registerInbound then attachFiltered for the conversationId; then starts a
      HandoffWorkflow with conversationId as the workflow id.
    * HandoffEvalScorer subscribed to ConversationEntity events; on RoutingDecided invokes
      HandoffJudge.score(filtered, decision) and calls ConversationEntity
      .recordHandoffScore(conversationId, score). On any other event type, no-op.
      Use componentClient — do NOT call the agent from a TimedAction.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation; uses
  Optional<T> for every nullable lifecycle field per Lesson 6). Table updater consumes
  ConversationEntity events. ONE query getAllConversations SELECT * AS conversations FROM
  conversation_view. No WHERE category or WHERE status filter (Akka cannot auto-index enum
  columns) — filter client-side in callers.

- 1 TimedAction ConversationSimulator — every 30s, reads next line from
  src/main/resources/sample-events/conversations.jsonl (loops at EOF) and calls
  ConversationQueue.receive with a fresh conversationId (UUID).

- 2 HttpEndpoints:
    * HandoffEndpoint at /api with GET /conversations (list from
      ConversationView.getAllConversations, filter client-side by ?category and ?status
      query params), GET /conversations/{id}, POST /conversations (body InboundTurn minus
      conversationId/receivedAt — server assigns), POST /conversations/{id}/unblock
      (body {decidedBy, note} — operator override: publishes the blocked conversation as
      RESOLVED with an audit note), GET /conversations/sse (serverSentEventsForView over
      getAllConversations), and three /api/metadata/{readme,risk-survey,eval-matrix}
      endpoints serving the YAML/MD files from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- HandoffTasks.java declaring the task constants: RESOLVE (resultConformsTo Reply.class,
  description "Resolve the conversation end-to-end and return a typed Reply").
- Domain records InboundTurn, FilteredContext, RoutingDecision, Reply, GuardrailVerdict,
  HandoffScore, and the Conversation entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9438 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/conversations.jsonl with 9 canned lines (3 billing-
  flavoured, 3 technical-flavoured, 2 unclear-flavoured, 1 designed to trip the routing
  guardrail with a prompt-injection signal).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii (MessageFilter),
  G1 guardrail before-agent-invocation (RoutingGuardrail), E1 eval-event on-decision-eval
  (HandoffJudge). Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  data.data_classes.pii = true, decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false, data.pii_handled_by_filter_before_llm = true,
  failure.failure_modes including "wrong-category-routing", "inappropriate-reply-content",
  "pii-leakage-via-llm", "prompt-injection-via-context", "refund-amount-fabrication";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/triage-agent.md, prompts/billing-specialist.md, prompts/technical-specialist.md,
  prompts/handoff-judge.md, prompts/routing-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: OpenAI Agents SDK Handoff Routing",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = conversation list with routing chip + status pill + score chip; centre = filtered
  context + routing decision + score; right = specialist reply + guardrail verdict +
  published reply or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: OpenAI Agents SDK Handoff Routing</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface with a per-agent
  dispatch on the agent class or Task<R> id. Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry pseudo-randomly per call
  (seeded by conversationId so reruns are deterministic), and deserialises into the agent's
  typed return.
- Per-agent mock-response shapes for THIS blueprint:
    triage-agent.json — 12 RoutingDecision entries spanning BILLING (charge disputes,
      refund requests, plan changes), TECHNICAL (API errors, integration questions,
      performance issues), and UNCLEAR (mixed content, very short messages, off-topic).
    routing-guardrail.json — 10 GuardrailVerdict entries. 7 with allowed=true and empty
      violations. 3 with allowed=false: "unredacted-email-in-context",
      "prompt-injection-signal", "out-of-scope-legal-request". The mock should match
      the trip entry in conversations.jsonl when called for the same conversationId.
    billing-specialist.json — 8 Reply entries: 5 INFO_PROVIDED or REFUND_INITIATED
      (within authority), 1 ESCALATED (refund > authority), 1 ARTICLE_LINKED, 1 designed
      to surface in testing.
    technical-specialist.json — 8 Reply entries: 5 INFO_PROVIDED or ARTICLE_LINKED,
      1 FOLLOW_UP_SCHEDULED, 1 ESCALATED, 1 edge case.
    handoff-judge.json — 10 HandoffScore entries, score 1–5, one-sentence rationale
      (category-correctness, confidence-calibration, reason-quality).
- A MockModelProvider.seedFor(conversationId) helper makes per-conversation selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent. BillingSpecialist and
  TechnicalSpecialist both extend akka.javasdk.agent.autonomous.AutonomousAgent and
  declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): filterConfirmStep 20s,
  guardrailStep 20s, triageStep 20s, billingStep / technicalStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Conversation is Optional<T>. The
  ConversationView row type uses the same Optional wrapping.
- (Lesson 7) HandoffTasks.java declares the RESOLVE Task<Reply> constant. Both specialists'
  definition().capability(TaskAcceptance.of(RESOLVE)...) reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9438 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour white, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. No "hidden" zombie panels.
- The MessageFilter runs INSIDE a Consumer before any LLM call — not inside an Agent's
  prompt and not after the LLM has seen the raw payload. This is the message-filter variant
  of the handoff primitive.
- The RoutingGuardrail fires BEFORE any specialist is invoked — before-agent-invocation,
  not before-agent-response. A blocked context never reaches a specialist.
- The HandoffEvalScorer Consumer reacts to RoutingDecided events and calls HandoffJudge via
  componentClient.forAgent(). It does NOT modify the workflow flow.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, use, use, marketing tone, competitor brand names.
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
