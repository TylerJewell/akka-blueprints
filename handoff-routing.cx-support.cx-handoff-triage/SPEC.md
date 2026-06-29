# SPEC — cx-handoff-triage

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Core Streaming Handoffs Customer Service.
**One-line pitch:** A triage agent classifies an inbound customer message and hands the conversation off to a sales or issues-and-repairs specialist that owns the response end-to-end, with PII redaction, a before-agent-response guardrail on every draft, and a before-tool-call guardrail that intercepts refund and order-fulfillment calls before they execute.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the conversation, then transfers ownership to a downstream specialist that produces the final response and may invoke tools. Three governance mechanisms layer on top:

- A **PII sanitizer** runs inside a Consumer between the raw inbound message event and the LLM call. The classifier and the specialists never see raw emails, phone numbers, customer names in greeting positions, or order ids.
- A **before-agent-response guardrail** runs on the specialist's draft response before it is streamed to the customer. It checks the draft against a policy rubric (no invented delivery windows, no discount amounts outside authority, no medical or legal advice, no echoing of `[REDACTED]` tokens) and blocks the draft when it violates, placing the conversation in `BLOCKED` for human review.
- A **before-tool-call guardrail** intercepts every refund or order-fulfillment tool invocation before execution. It validates that the request is within the specialist's authority (refund amount ≤ policy ceiling, order status matches a valid fulfillment state) and cancels the tool call when it does not, returning a rejection signal to the specialist.

The pattern is a textbook fan-out-of-one: the workflow branches on the classifier's category, and only the chosen specialist receives traffic for that conversation.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live conversation list. Each conversation displays its category chip, status pill, and (if resolved) the published specialist response.
2. `ConversationSimulator` (TimedAction) ticks every 30 s and inserts a new canned opening message from `sample-events/customer-conversations.jsonl` into `ConversationQueue`.
3. For each new message: `PiiSanitizer` (Consumer) redacts the payload, registers a `ConversationEntity`, and starts a `ConversationWorkflow`.
4. The workflow calls `TriageAgent`, gets a `TriageDecision { category, confidence, reason }`, and emits `ConversationTriaged` on the entity.
5. Branch on `category`:
   - `SALES` → workflow calls `SalesSpecialist` with the `RESOLVE` task and waits for the typed `SpecialistResponse` result.
   - `ISSUES_REPAIRS` → workflow calls `IssuesRepairsSpecialist` with the same `RESOLVE` task.
   - `UNCLEAR` → workflow emits `ConversationEscalated`; ends.
6. When the specialist invokes a refund or fulfillment tool, `ToolCallGuardrail` intercepts. On `allowed=false` the tool call is cancelled and the specialist receives a policy-rejection. On `allowed=true` the tool executes normally.
7. The specialist's draft `SpecialistResponse` passes through `ResponseGuardrail`. If accepted, `ResponsePublished` is emitted (terminal `RESOLVED`). If rejected, `ResponseBlocked` is emitted (terminal `BLOCKED`) with the violation list.
8. The user can click any conversation card and see the redacted opening message, the triage decision, the specialist's draft (or blocked draft + violations), and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationSimulator` | `TimedAction` | Drips simulated customer conversations into `ConversationQueue` every 30 s. | scheduler | `ConversationQueue` |
| `ConversationQueue` | `EventSourcedEntity` | Append-only audit log of every inbound message (`InboundMessageReceived`). | `ConversationSimulator`, `ChatEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `ConversationQueue` events; redacts PII; registers `ConversationEntity`; starts a `ConversationWorkflow`. | `ConversationQueue` events | `ConversationEntity`, `ConversationWorkflow` |
| `TriageAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedMessage` into `SALES` / `ISSUES_REPAIRS` / `UNCLEAR` with confidence + reason. | invoked by `ConversationWorkflow` | returns `TriageDecision` |
| `SalesSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for sales conversations. Returns typed `SpecialistResponse`. May invoke refund/fulfillment tools subject to `ToolCallGuardrail`. | invoked by `ConversationWorkflow` | returns `SpecialistResponse` |
| `IssuesRepairsSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for issues and repairs conversations. Returns typed `SpecialistResponse`. May invoke return/replacement tools subject to `ToolCallGuardrail`. | invoked by `ConversationWorkflow` | returns `SpecialistResponse` |
| `ResponseGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `SpecialistResponse` against the policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `ConversationWorkflow` | returns `GuardrailVerdict` |
| `ToolCallGuardrail` | `Agent` (typed) | Before-tool-call guardrail: validates a `ToolCallRequest` against tool-policy rules before execution. Returns `ToolCallVerdict { allowed, reason }`. | invoked inline before each tool execution | returns `ToolCallVerdict` |
| `ConversationWorkflow` | `Workflow` | Per-conversation orchestration: triage → branch → resolve (with tool guardrails) → response guardrail → publish. | `PiiSanitizer` (start) | `ConversationEntity` |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle and history. | `ConversationWorkflow`, `PiiSanitizer` | `ConversationView` |
| `ConversationView` | `View` | Read-model row per conversation. | `ConversationEntity` events | `ChatEndpoint` |
| `ChatEndpoint` | `HttpEndpoint` | `/api/conversations/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `ConversationView`, `ConversationEntity`, `ConversationQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundMessage(
    String conversationId,
    String channel,             // "chat" | "email" | "voice-transcript"
    String openingMessage,
    String customerHandle,      // opaque customer identifier from the channel
    Instant receivedAt
) {}

record SanitizedMessage(
    String redactedText,
    List<String> piiCategoriesFound
) {}

enum ConversationCategory { SALES, ISSUES_REPAIRS, UNCLEAR }

record TriageDecision(
    ConversationCategory category,
    String confidence,          // "high" | "medium" | "low"
    String reason               // one short sentence
) {}

enum ResponseAction {
    ORDER_PLACED, REFUND_INITIATED, REPLACEMENT_ARRANGED,
    INFO_PROVIDED, FOLLOW_UP_SCHEDULED, ESCALATED
}

record SpecialistResponse(
    String responseText,
    ResponseAction action,
    String specialistTag,       // "sales" | "issues-repairs"
    Instant resolvedAt
) {}

record ToolCallRequest(
    String toolName,            // "processRefund" | "placeOrder" | "scheduleReplacement"
    String conversationId,
    Map<String, Object> parameters
) {}

record ToolCallVerdict(
    boolean allowed,
    String reason               // empty when allowed; violation token when denied
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,    // empty when allowed
    String rubricVersion
) {}

record Conversation(
    String conversationId,
    InboundMessage incoming,
    Optional<SanitizedMessage> sanitized,
    Optional<TriageDecision> triage,
    Optional<SpecialistResponse> response,
    Optional<GuardrailVerdict> guardrail,
    Optional<String> escalationReason,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ConversationStatus {
    RECEIVED,
    SANITIZED,
    TRIAGED,
    ROUTED_SALES,
    ROUTED_ISSUES_REPAIRS,
    RESPONSE_DRAFTED,
    TOOL_CALL_BLOCKED,
    BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `ConversationEntity`: `ConversationRegistered`, `ConversationSanitized`, `ConversationTriaged`, `ConversationRouted`, `ResponseDrafted`, `ToolCallBlocked`, `GuardrailVerdictAttached`, `ResponsePublished`, `ResponseBlocked`, `ConversationEscalated`.

Events on `ConversationQueue`: `InboundMessageReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/conversations` — list all conversations (newest-first), optional `?category=SALES|ISSUES_REPAIRS|UNCLEAR&status=…` filtered client-side.
- `GET /api/conversations/{id}` — one conversation.
- `POST /api/conversations` — manually submit a message (body `InboundMessage` minus `conversationId` and `receivedAt`); server assigns both.
- `POST /api/conversations/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator approves publishing the blocked draft.
- `GET /api/conversations/sse` — Server-Sent Events for every conversation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Core Streaming Handoffs Customer Service</title>`.

The App UI tab is a three-pane layout: **left** is the conversation list (status pill + category chip), **centre** is the selected conversation's redacted opening message + triage decision, **right** is the chosen specialist's draft + guardrail verdict + published response (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** on `SalesSpecialist` and `IssuesRepairsSpecialist`: checks every draft `SpecialistResponse` against a rubric (no invented delivery windows, no discount amounts outside authority, no medical or legal advice, no echoing of `[REDACTED]` tokens). Blocking — a violation puts the conversation in `BLOCKED` for human review.
- **G2 — before-tool-call guardrail** on any refund or order-fulfillment tool invocation: validates the call against tool-policy rules (refund ceiling, valid fulfillment state) before the tool executes. Blocking — a denied call is cancelled and the specialist receives a rejection signal.
- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts emails, phone numbers, names, and order ids from the opening message before any LLM sees it. The categories found are kept for audit; only redacted text reaches the agents.

## 9. Agent prompts

- `TriageAgent` → `prompts/triage-agent.md`. Typed classifier; returns one of `SALES`, `ISSUES_REPAIRS`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `SalesSpecialist` → `prompts/sales-specialist.md`. Owns the `RESOLVE` task for sales conversations. Never invents pricing or discount amounts; routes outside its authority to `ESCALATED`.
- `IssuesRepairsSpecialist` → `prompts/issues-repairs-specialist.md`. Owns the `RESOLVE` task for issues and repairs conversations. Never invents repair timelines; cites a published policy document when applicable.
- `ResponseGuardrail` → `prompts/response-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.
- `ToolCallGuardrail` → `prompts/tool-call-guardrail.md`. Returns a `ToolCallVerdict { allowed, reason }`. Evaluates tool name, parameters, and conversation context against tool-policy rules.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a sales conversation → triaged `SALES` → resolved by `SalesSpecialist` → response guardrail passes → published.
2. **J2** — Simulator drips an issues-and-repairs conversation → triaged `ISSUES_REPAIRS` → resolved by `IssuesRepairsSpecialist` → response guardrail passes → published.
3. **J3** — An ambiguous opening message triages as `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
4. **J4** — A draft that promises next-day delivery (outside policy) is blocked by the response guardrail; the conversation lands in `BLOCKED`; the operator can unblock or leave it.
5. **J5** — A refund tool call that exceeds the specialist's authority ceiling is cancelled by the tool-call guardrail; the specialist pivots and returns an `ESCALATED` response.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named cx-handoff-triage demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real channel integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-cx-handoff-triage.
Java package io.akka.samples.corestreaminghandoffscustomerservice. Akka 3.6.0. HTTP port 9119.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) TriageAgent — classifier. System prompt loaded from
  prompts/triage-agent.md. Input: SanitizedMessage{redactedText, piiCategoriesFound:
  List<String>}. Output: TriageDecision{category: ConversationCategory
  (SALES/ISSUES_REPAIRS/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent SalesSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(4)). System prompt from prompts/sales-specialist.md. Input:
  SanitizedMessage + TriageDecision. Output: SpecialistResponse{responseText, action:
  ResponseAction, specialistTag = "sales", resolvedAt}. Never invents pricing or discount
  amounts; sets action=ESCALATED with a reason when outside authority.

- 1 AutonomousAgent IssuesRepairsSpecialist — definition() with
  capability(TaskAcceptance.of(RESOLVE).maxIterationsPerTask(4)). System prompt from
  prompts/issues-repairs-specialist.md. Same input shape; specialistTag = "issues-repairs".

- 1 Agent (typed) ResponseGuardrail — typed rubric check. System prompt from
  prompts/response-guardrail.md. Input: SanitizedMessage + SpecialistResponse. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by ConversationWorkflow before publishing; blocking.

- 1 Agent (typed) ToolCallGuardrail — before-tool-call policy check. System prompt from
  prompts/tool-call-guardrail.md. Input: ToolCallRequest{toolName, conversationId,
  parameters: Map<String,Object>}. Output: ToolCallVerdict{allowed: boolean, reason: String}.
  Evaluated synchronously before each tool execution inside the specialist's task loop;
  blocking on allowed=false (the tool call is cancelled).

- 1 Workflow ConversationWorkflow per conversationId. Steps:
    triageStep -> routeStep -> {salesStep | issuesStep | escalateStep}
               -> guardrailStep -> publishStep
  triageStep calls componentClient.forAgent().inSession(conversationId)
    .method(TriageAgent::triage).invoke(sanitized). On success emits ConversationTriaged
    via ConversationEntity.recordTriage.
  routeStep branches on TriageDecision.category:
    SALES -> proceed to salesStep (emits ConversationRouted{SALES})
    ISSUES_REPAIRS -> proceed to issuesStep (emits ConversationRouted{ISSUES_REPAIRS})
    UNCLEAR -> escalateStep (emits ConversationEscalated; terminates).
  salesStep / issuesStep call forAutonomousAgent(<Specialist>.class, conversationId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, triage))) returning a taskId,
    then forTask(taskId).result(ChatTasks.RESOLVE) to block on the typed SpecialistResponse.
    Tool calls within the task invoke ToolCallGuardrail before execution; on allowed=false
    the call is cancelled, ToolCallBlocked is emitted, and the specialist receives a rejection.
    On success emits ResponseDrafted.
  guardrailStep calls forAgent(...).method(ResponseGuardrail::check).invoke(sanitized, draft).
    On verdict.allowed=true proceed to publishStep (emits ResponsePublished, terminal RESOLVED).
    On verdict.allowed=false emit ResponseBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on triageStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on salesStep, issuesStep, and
    publishStep. defaultStepRecovery(maxRetries(2).failoverTo(ConversationWorkflow::error)).

- 2 EventSourcedEntities:
    * ConversationQueue — append-only audit log. Command receive(InboundMessage) emits
      InboundMessageReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.conversationId.
    * ConversationEntity (one per conversationId) — full per-conversation lifecycle. State
      Conversation{conversationId, incoming: InboundMessage, Optional<SanitizedMessage>
      sanitized, Optional<TriageDecision> triage, Optional<SpecialistResponse> response,
      Optional<GuardrailVerdict> guardrail, Optional<String> escalationReason,
      ConversationStatus status, Instant createdAt, Optional<Instant> finishedAt}.
      ConversationStatus enum: RECEIVED, SANITIZED, TRIAGED, ROUTED_SALES,
      ROUTED_ISSUES_REPAIRS, RESPONSE_DRAFTED, TOOL_CALL_BLOCKED, BLOCKED, RESOLVED,
      ESCALATED.
      Events: ConversationRegistered, ConversationSanitized, ConversationTriaged,
      ConversationRouted, ResponseDrafted, ToolCallBlocked, GuardrailVerdictAttached,
      ResponsePublished, ResponseBlocked, ConversationEscalated.
      Commands: registerIncoming, attachSanitized, recordTriage, recordRouting, recordDraft,
      recordToolCallBlock, recordGuardrailVerdict, publish, block, escalate, unblock,
      getConversation. emptyState() returns Conversation.initial("") with no
      commandContext() reference.

- 1 Consumer PiiSanitizer subscribed to ConversationQueue events; for each
  InboundMessageReceived applies a regex+heuristic redaction pipeline (emails, phone
  numbers, order ids matching ORD-\d+, customer handles in greeting positions) to
  openingMessage, builds SanitizedMessage with piiCategoriesFound, and calls
  ConversationEntity.registerIncoming then attachSanitized for the conversationId; then
  starts a ConversationWorkflow with conversationId as the workflow id.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation; uses
  Optional<T> for every nullable lifecycle field per Lesson 6). Table updater consumes
  ConversationEntity events. ONE query getAllConversations SELECT * AS conversations FROM
  conversation_view. No WHERE category or WHERE status filter — filter client-side.

- 1 TimedAction ConversationSimulator — every 30s, reads next line from
  src/main/resources/sample-events/customer-conversations.jsonl (loops at EOF) and calls
  ConversationQueue.receive with a fresh conversationId (UUID).

- 2 HttpEndpoints:
    * ChatEndpoint at /api with GET /conversations (list from
      ConversationView.getAllConversations, filter client-side by ?category and ?status
      query params), GET /conversations/{id}, POST /conversations (body InboundMessage
      minus conversationId/receivedAt — server assigns), POST /conversations/{id}/unblock
      (body {decidedBy, note} — operator override: publishes the blocked draft as RESOLVED
      with an audit note), GET /conversations/sse (serverSentEventsForView over
      getAllConversations), and three /api/metadata/{readme,risk-survey,eval-matrix}
      endpoints serving the YAML/MD files from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ChatTasks.java declaring the task constants: RESOLVE (resultConformsTo
  SpecialistResponse.class, description "Resolve the customer conversation end-to-end and
  return a typed SpecialistResponse").
- Domain records InboundMessage, SanitizedMessage, TriageDecision, SpecialistResponse,
  ToolCallRequest, ToolCallVerdict, GuardrailVerdict, and the Conversation entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9119 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/customer-conversations.jsonl with 9 canned lines (3
  SALES-flavoured, 3 ISSUES_REPAIRS-flavoured, 2 UNCLEAR-flavoured, 1 designed to trip the
  response guardrail with a next-day delivery promise).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls: G1 guardrail before-agent-response,
  G2 guardrail before-tool-call, S1 sanitizer pii. Matching simplified_view list.
  No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  data.data_classes.pii = true, decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false, data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-category-routing", "inappropriate-reply-content",
  "pii-leakage-via-llm", "out-of-authority-tool-call", "invented-delivery-window";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/triage-agent.md, prompts/sales-specialist.md,
  prompts/issues-repairs-specialist.md, prompts/response-guardrail.md,
  prompts/tool-call-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Core Streaming Handoffs Customer
  Service", prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = conversation list with category chip + status pill; centre = redacted opening
  message + triage block; right = specialist draft + guardrail verdict + published response
  or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Core Streaming Handoffs Customer Service</title>.
  No subtitle on the Overview tab.

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
  pseudo-randomly per call (seeded by conversationId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    triage-agent.json — 12 TriageDecision entries spanning SALES (product
      questions, upgrade requests, new order placement), ISSUES_REPAIRS
      (defect reports, return requests, repair status questions), and UNCLEAR
      (very short messages, off-topic, mixed content). Confidence + a
      one-sentence reason on each.
    sales-specialist.json — 8 SpecialistResponse entries: 4 with action
      INFO_PROVIDED or ORDER_PLACED (within authority), 1 with FOLLOW_UP_SCHEDULED,
      1 with action ESCALATED (discount above ceiling), 1 with ORDER_PLACED plus
      a tool call that will pass the tool guardrail, 1 designed to trip the
      response guardrail (promises next-day delivery outside policy).
    issues-repairs-specialist.json — 8 SpecialistResponse entries: 4 with
      INFO_PROVIDED or REPLACEMENT_ARRANGED, 1 with FOLLOW_UP_SCHEDULED,
      1 with ESCALATED, 1 with REFUND_INITIATED plus a tool call that will
      pass the tool guardrail, 1 with a refund amount that exceeds the policy
      ceiling (to trip the tool guardrail).
    response-guardrail.json — 10 GuardrailVerdict entries. 7 with allowed=true
      and empty violations. 3 with allowed=false: "invented-delivery-window",
      "out-of-authority-discount", "echoes-redacted-token".
    tool-call-guardrail.json — 8 ToolCallVerdict entries. 5 with allowed=true.
      3 with allowed=false: "refund-exceeds-ceiling", "invalid-fulfillment-state",
      "missing-required-parameter".
- A MockModelProvider.seedFor(conversationId) helper makes per-conversation
  selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  SalesSpecialist and IssuesRepairsSpecialist both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): triageStep 20s,
  guardrailStep 20s, salesStep / issuesStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Conversation is Optional<T>. The
  ConversationView row type uses the same Optional wrapping.
- (Lesson 7) ChatTasks.java declares the RESOLVE Task<SpecialistResponse> constant.
  Both specialists' definition().capability(TaskAcceptance.of(RESOLVE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9119 in application.conf; not 9000.
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
- The PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside
  an Agent's prompt and not after the LLM has seen the raw payload.
- The ToolCallGuardrail is invoked synchronously before each tool execution.
  A cancelled tool call emits ToolCallBlocked on ConversationEntity and passes
  a policy-rejection signal back to the specialist within the same task loop
  iteration.
- The response guardrail step happens BEFORE ResponsePublished. A blocked draft
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
