# SPEC — discord-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Discord AI-Powered Bot.
**One-line pitch:** A classifier agent reads an inbound Discord message, routes it to a community or technical specialist that owns the reply end-to-end, with PII sanitization before any model call and a before-agent-response guardrail checking every draft reply against a community-policy rubric.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which specialist* should own the reply, then transfers the full message context to that specialist so it can produce the final output without further delegation. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw Discord message event and any LLM call. The classifier and the specialists never see raw email addresses, phone numbers, or Discord user ids.
- A **before-agent-response guardrail** runs on every draft reply produced by either specialist. It checks the draft against a community-policy rubric (no unapproved external links, no impersonation of official Akka staff, no echoing of `[REDACTED]` tokens, no off-topic promotional content) and blocks the draft from reaching the publish step if it violates.

An **on-decision eval** fires every time the classifier emits a routing decision. A `RoutingJudge` agent grades the decision on a 1–5 rubric and writes the score back to the `MessageEntity` as out-of-band metadata.

The pattern is a fan-out-of-one: the workflow branches on the classifier's category, and only the chosen specialist is invoked. The unchosen specialist sees no traffic for that message.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live message list. Every message card displays its category chip, status pill, routing score, and (if published) the bot's reply.
2. `MessageSimulator` (TimedAction) ticks every 30 s and inserts a new canned Discord message from `sample-events/discord-messages.jsonl` into `MessageQueue`.
3. For each new message: `PiiSanitizer` (Consumer) redacts the payload, registers a `MessageEntity`, and starts a `DiscordWorkflow`.
4. The workflow calls `ClassifierAgent`, receives a `RoutingDecision { category, confidence, reason }`, and emits `MessageClassified` on the entity.
5. Branch on `category`:
   - `COMMUNITY` → workflow calls `CommunitySpecialist` with the `REPLY` task and waits for the typed `BotReply` result.
   - `TECHNICAL` → workflow calls `TechnicalSpecialist` with the same `REPLY` task.
   - `UNCLEAR` → workflow emits `MessageEscalated`; ends.
6. The specialist's draft `BotReply` passes through the before-agent-response guardrail. If accepted, `ReplyPublished` is emitted (terminal `PUBLISHED`). If rejected, `ReplyBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `MessageClassified` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the entity.
8. The user can click any message card and see the redacted message, the routing reason, the routing score, the chosen specialist, the draft (or blocked draft + violations), and the published reply.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MessageSimulator` | `TimedAction` | Drips simulated Discord messages into `MessageQueue` every 30 s. | scheduler | `MessageQueue` |
| `MessageQueue` | `EventSourcedEntity` | Append-only audit log of every inbound message (`InboundMessageReceived`). | `MessageSimulator`, `DiscordEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `MessageQueue` events; redacts PII; registers `MessageEntity`; starts a `DiscordWorkflow`. | `MessageQueue` events | `MessageEntity`, `DiscordWorkflow` |
| `ClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedMessage` into `COMMUNITY` / `TECHNICAL` / `UNCLEAR` with confidence + reason. | invoked by `DiscordWorkflow` | returns `RoutingDecision` |
| `CommunitySpecialist` | `AutonomousAgent` | Owns the `REPLY` task for community messages. Returns typed `BotReply`. | invoked by `DiscordWorkflow` | returns `BotReply` |
| `TechnicalSpecialist` | `AutonomousAgent` | Owns the `REPLY` task for technical messages. Returns typed `BotReply`. | invoked by `DiscordWorkflow` | returns `BotReply` |
| `RoutingJudge` | `Agent` (typed) | Grades a routing decision against the sanitized message. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `ReplyGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `BotReply` against the community-policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `DiscordWorkflow` | returns `GuardrailVerdict` |
| `DiscordWorkflow` | `Workflow` | Per-message orchestration: classify → branch → reply → guardrail → publish. | `PiiSanitizer` (start) | `MessageEntity` |
| `MessageEntity` | `EventSourcedEntity` | Per-message lifecycle. | `DiscordWorkflow`, `RoutingEvalScorer` | `MessageView` |
| `MessageView` | `View` | Read-model row per message. | `MessageEntity` events | `DiscordEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `MessageEntity` events; on `MessageClassified` invokes `RoutingJudge` and writes `RoutingScored` back. | `MessageEntity` events | `MessageEntity` |
| `DiscordEndpoint` | `HttpEndpoint` | `/api/messages/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `MessageView`, `MessageEntity`, `MessageQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingMessage(
    String messageId,
    String guildId,
    String channelName,        // "general" | "help" | "announcements" | "tech-support"
    String authorHandle,       // raw Discord handle — PII
    String content,
    Instant receivedAt
) {}

record SanitizedMessage(
    String redactedContent,
    String channelName,        // retained; not PII
    List<String> piiCategoriesFound
) {}

enum MessageCategory { COMMUNITY, TECHNICAL, UNCLEAR }

record RoutingDecision(
    MessageCategory category,
    String confidence,        // "high" | "medium" | "low"
    String reason             // one short sentence
) {}

enum ReplyAction { ANSWERED, DOCS_LINKED, FOLLOW_UP_REQUESTED, ACKNOWLEDGED, ESCALATED }

record BotReply(
    String replyBody,
    ReplyAction action,
    String specialistTag,     // "community" | "technical"
    Instant repliedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,  // empty when allowed
    String rubricVersion
) {}

record RoutingScore(
    int score,                // 1..5
    String rationale,
    Instant scoredAt
) {}

record Message(
    String messageId,
    IncomingMessage incoming,
    Optional<SanitizedMessage> sanitized,
    Optional<RoutingDecision> routing,
    Optional<BotReply> reply,
    Optional<GuardrailVerdict> guardrail,
    Optional<RoutingScore> routingScore,
    Optional<String> escalationReason,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED,
    SANITIZED,
    CLASSIFIED,
    ROUTED_COMMUNITY,
    ROUTED_TECHNICAL,
    REPLY_DRAFTED,
    BLOCKED,
    PUBLISHED,
    ESCALATED
}
```

Events on `MessageEntity`: `MessageRegistered`, `MessageSanitized`, `MessageClassified`, `MessageRouted`, `ReplyDrafted`, `GuardrailVerdictAttached`, `ReplyPublished`, `ReplyBlocked`, `MessageEscalated`, `RoutingScored`.

Events on `MessageQueue`: `InboundMessageReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/messages` — list all messages (newest-first), optional `?category=COMMUNITY|TECHNICAL|UNCLEAR&status=…` filtered client-side.
- `GET /api/messages/{id}` — one message.
- `POST /api/messages` — manually submit a message (body `IncomingMessage` minus `messageId` and `receivedAt`); server assigns both.
- `POST /api/messages/{id}/unblock` — body `{ decidedBy, note }` — moderator override; transitions `BLOCKED` to `PUBLISHED` if the moderator approves the blocked draft.
- `GET /api/messages/sse` — Server-Sent Events for every message change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Discord AI-Powered Bot</title>`.

The App UI tab is a three-pane layout: **left** is the message list (status pill + category chip + score chip), **centre** is the selected message's redacted content + routing decision + score, **right** is the chosen specialist's draft + guardrail verdict + published reply (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts email addresses, phone numbers, Discord user ids (numeric snowflakes), and author handles from the message content before any LLM sees it. The categories found are kept for audit; the redacted text is what reaches the agents.
- **G1 — before-agent-response guardrail** on `CommunitySpecialist` and `TechnicalSpecialist`: checks every draft `BotReply` against a rubric (no unapproved external links, no impersonation of official staff, no `[REDACTED]` echoes, no off-topic promotional content). Blocking — a violation puts the message in `BLOCKED` for moderator review.

## 9. Agent prompts

- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier; returns one of `COMMUNITY`, `TECHNICAL`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `CommunitySpecialist` → `prompts/community-specialist.md`. Owns the `REPLY` task for community messages. Warm, concise tone; escalates hate speech immediately.
- `TechnicalSpecialist` → `prompts/technical-specialist.md`. Owns the `REPLY` task for technical messages. Links to docs when applicable; never invents a docs URL.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a routing decision on a 1–5 rubric with one-sentence rationale.
- `ReplyGuardrail` → `prompts/reply-guardrail.md`. Returns `GuardrailVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a community-flavoured message → classified `COMMUNITY` → replied by `CommunitySpecialist` → guardrail passes → published.
2. **J2** — Simulator drips a technical message → classified `TECHNICAL` → replied by `TechnicalSpecialist` → guardrail passes → published.
3. **J3** — An ambiguous message classifies as `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
4. **J4** — A draft that links to an unapproved external URL is blocked; the message lands in `BLOCKED` with the violation listed; the moderator can either unblock or leave it.
5. **J5** — Every classified message carries a `RoutingScore` (1–5) and rationale within ~10 s of the classification event.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named discord-router demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real Discord integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-discord-router.
Java package io.akka.samples.discordaipoweredbot. Akka 3.6.0. HTTP port 9452.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClassifierAgent — classifier. System prompt loaded from
  prompts/classifier-agent.md. Input: SanitizedMessage{redactedContent, channelName,
  piiCategoriesFound: List<String>}. Output: RoutingDecision{category: MessageCategory
  (COMMUNITY/TECHNICAL/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent CommunitySpecialist — definition() with capability(TaskAcceptance.of(REPLY)
  .maxIterationsPerTask(3)). System prompt from prompts/community-specialist.md. Input:
  SanitizedMessage + RoutingDecision. Output: BotReply{replyBody, action: ReplyAction,
  specialistTag = "community", repliedAt}. Never exceeds its mandate; sets
  action=ESCALATED for hate speech or harassment.

- 1 AutonomousAgent TechnicalSpecialist — definition() with capability(TaskAcceptance.of(REPLY)
  .maxIterationsPerTask(3)). System prompt from prompts/technical-specialist.md. Same
  input shape; specialistTag = "technical".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  SanitizedMessage + RoutingDecision. Output: RoutingScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Agent (typed) ReplyGuardrail — typed rubric check. System prompt from
  prompts/reply-guardrail.md. Input: SanitizedMessage + BotReply. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by DiscordWorkflow before publishing; blocking.

- 1 Workflow DiscordWorkflow per messageId. Steps:
    classifyStep -> routeStep -> {communityStep | technicalStep | escalateStep}
                -> guardrailStep -> publishStep
  classifyStep calls componentClient.forAgent().inSession(messageId).method(ClassifierAgent::classify)
    .invoke(sanitized). On success emits MessageClassified via MessageEntity.recordClassification.
  routeStep branches on RoutingDecision.category:
    COMMUNITY -> proceed to communityStep (emits MessageRouted{COMMUNITY})
    TECHNICAL -> proceed to technicalStep (emits MessageRouted{TECHNICAL})
    UNCLEAR -> escalateStep (emits MessageEscalated; terminates).
  communityStep / technicalStep call forAutonomousAgent(<Specialist>.class, messageId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, routing))) returning a taskId,
    then forTask(taskId).result(DiscordTasks.REPLY) to block on the typed BotReply.
    On success emits ReplyDrafted.
  guardrailStep calls forAgent(...).method(ReplyGuardrail::check).invoke(sanitized, draft).
    On verdict.allowed=true proceed to publishStep (emits ReplyPublished, terminal
    PUBLISHED). On verdict.allowed=false emit ReplyBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on communityStep, technicalStep, and
    publishStep. defaultStepRecovery(maxRetries(2).failoverTo(DiscordWorkflow::error)).

- 2 EventSourcedEntities:
    * MessageQueue — append-only audit log. Command receive(IncomingMessage) emits
      InboundMessageReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.messageId.
    * MessageEntity (one per messageId) — full per-message lifecycle. State Message{messageId,
      incoming: IncomingMessage, Optional<SanitizedMessage> sanitized,
      Optional<RoutingDecision> routing, Optional<BotReply> reply,
      Optional<GuardrailVerdict> guardrail, Optional<RoutingScore> routingScore,
      Optional<String> escalationReason, MessageStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. MessageStatus enum: RECEIVED, SANITIZED, CLASSIFIED,
      ROUTED_COMMUNITY, ROUTED_TECHNICAL, REPLY_DRAFTED, BLOCKED, PUBLISHED, ESCALATED.
      Events: MessageRegistered, MessageSanitized, MessageClassified, MessageRouted,
      ReplyDrafted, GuardrailVerdictAttached, ReplyPublished, ReplyBlocked,
      MessageEscalated, RoutingScored. Commands: registerIncoming, attachSanitized,
      recordClassification, recordRouting, recordDraft, recordGuardrailVerdict, publish,
      block, escalate, recordRoutingScore, unblock, getMessage. emptyState() returns
      Message.initial("") with no commandContext() reference.

- 2 Consumers:
    * PiiSanitizer subscribed to MessageQueue events; for each InboundMessageReceived
      applies a regex+heuristic redaction pipeline (emails, phone numbers, Discord user-id
      snowflakes matching \d{17,19}, author handles stripped to [REDACTED-HANDLE]) to
      content, builds SanitizedMessage with piiCategoriesFound, and calls MessageEntity
      .registerIncoming then attachSanitized for the messageId; then starts a DiscordWorkflow
      with messageId as the workflow id.
    * RoutingEvalScorer subscribed to MessageEntity events; on MessageClassified invokes
      RoutingJudge.score(sanitized, decision) and calls MessageEntity.recordRoutingScore(
      messageId, score). On any other event type, no-op. Use componentClient — do NOT
      call the agent from a TimedAction.

- 1 View MessageView with row type MessageRow (mirrors Message; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes MessageEntity events.
  ONE query getAllMessages SELECT * AS messages FROM message_view. No WHERE category or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction MessageSimulator — every 30s, reads next line from
  src/main/resources/sample-events/discord-messages.jsonl (loops at EOF) and calls
  MessageQueue.receive with a fresh messageId (UUID).

- 2 HttpEndpoints:
    * DiscordEndpoint at /api with GET /messages (list from MessageView.getAllMessages,
      filter client-side by ?category and ?status query params), GET /messages/{id},
      POST /messages (body IncomingMessage minus messageId/receivedAt — server assigns),
      POST /messages/{id}/unblock (body {decidedBy, note} — moderator override:
      publishes the blocked draft as PUBLISHED with an audit note),
      GET /messages/sse (serverSentEventsForView over getAllMessages), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DiscordTasks.java declaring the task constants: REPLY (resultConformsTo BotReply.class,
  description "Reply to the Discord message end-to-end and return a typed BotReply").
- Domain records IncomingMessage, SanitizedMessage, RoutingDecision, BotReply,
  GuardrailVerdict, RoutingScore, and the Message entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9452 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/discord-messages.jsonl with 9 canned lines (3 COMMUNITY-
  flavoured, 3 TECHNICAL-flavoured, 2 UNCLEAR-flavoured, 1 designed to trip the guardrail
  with an unapproved external link in the draft reply).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii,
  G1 guardrail before-agent-response community-policy. Matching simplified_view list.
  No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = discord-community-moderation,
  data.data_classes.pii = true, decisions.authority_level = draft-only,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  blocked drafts wait for a moderator), data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-category-routing", "unapproved-link-in-reply",
  "pii-leakage-via-llm", "impersonation-of-staff", "off-topic-promotion"; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/classifier-agent.md, prompts/community-specialist.md, prompts/technical-specialist.md,
  prompts/routing-judge.md, prompts/reply-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Discord AI-Powered Bot", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = message list with category chip + status pill + score chip; centre = redacted
  content + routing decision block; right = specialist draft + guardrail verdict + published
  reply or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Discord AI-Powered Bot</title>. No subtitle on the Overview tab.

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
    classifier-agent.json — 12 RoutingDecision entries spanning COMMUNITY
      (onboarding questions, general feedback, event announcements),
      TECHNICAL (SDK errors, API questions, deployment issues), and UNCLEAR
      (very short messages, emoji-only, mixed-intent). Confidence + a
      one-sentence reason on each.
    community-specialist.json — 8 BotReply entries: 5 with action
      ANSWERED or ACKNOWLEDGED (within community specialist scope), 1 with
      action FOLLOW_UP_REQUESTED, 1 with action ESCALATED (hate speech
      detected), 1 designed to trip the guardrail (links to an unapproved
      external site) so guardrail tests have material.
    technical-specialist.json — 8 BotReply entries: 5 with ANSWERED or
      DOCS_LINKED, 1 with FOLLOW_UP_REQUESTED, 1 with ESCALATED,
      1 designed to trip the guardrail (impersonates an official Akka staff
      member — for the impersonation rule).
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence
      rationale matching the rubric (category-correctness / confidence-
      calibration / reason-quality).
    reply-guardrail.json — 10 GuardrailVerdict entries. 7 with
      allowed=true and empty violations. 3 with allowed=false and one
      violation each ("unapproved-external-link", "staff-impersonation",
      "echoes-redacted-token"). The mock should match the rubric-tripping
      entries above when called for the same messageId.
- A MockModelProvider.seedFor(messageId) helper makes per-message selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  CommunitySpecialist and TechnicalSpecialist both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): classifyStep 20s,
  guardrailStep 20s, communityStep / technicalStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Message is Optional<T>. The
  MessageView row type uses the same Optional wrapping.
- (Lesson 7) DiscordTasks.java declares the REPLY Task<BotReply> constant.
  Both specialists' definition().capability(TaskAcceptance.of(REPLY)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9452 in application.conf; not 9000.
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
- The RoutingEvalScorer Consumer reacts to MessageClassified events and calls
  RoutingJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- The guardrail step happens BEFORE ReplyPublished. A blocked draft never
  reaches the UI as published — only as a "blocked draft + violations"
  surface for the moderator.
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
