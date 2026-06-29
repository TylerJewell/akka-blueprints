# SPEC — inbox-classifier

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** InboxZero Lite — Email Classifier.
**One-line pitch:** A classifier agent reads each inbound email, assigns it one of four priority labels, and hands it off to a routing agent that takes the appropriate action, with PII redaction before any LLM call, a before-tool-call guardrail on destructive actions, and an inline eval scoring every classification decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *how* a message should be handled, then transfers ownership to a routing agent that executes the action end-to-end. The routing agent is responsible for the full disposition; the classifier does not direct execution. Three governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw inbound message event and the LLM call. The classifier and the routing agent never see raw email addresses, phone numbers, national ID patterns, or account numbers in message bodies.
- A **before-tool-call guardrail** runs before any destructive action executes (`MOVE_TO_SPAM`, `DELETE`). It verifies the action is warranted given the sanitized message content and the classification, blocking the action when it is not.
- An **on-decision eval** fires every time the classifier emits a label. A `ClassificationJudge` agent grades the decision against the sanitized payload on a 1–5 rubric. The score and rationale are written back to the message entity and surfaced in the UI.

The pattern is a textbook fan-out-of-one: the classifier picks the label, and the routing agent owns the action for that label. The routing agent sees no other labels' traffic.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live message list. Every message displays its label chip, status pill, classification score, and (if actioned) the routing decision.
2. `InboxSimulator` (TimedAction) ticks every 30 s and inserts a new canned email message from `sample-events/inbox-messages.jsonl` into `InboxQueue`.
3. For each new message: `BodySanitizer` (Consumer) redacts the body, registers a `MessageEntity`, and starts a `RoutingWorkflow`.
4. The workflow calls `ClassifierAgent`, gets a `ClassificationDecision { label, confidence, reason }`, and emits `MessageClassified` on the entity.
5. The workflow calls `RoutingAgent` with the `ROUTE` task and the classification. The agent returns a typed `RoutingDecision { action, targetFolder, forwardAddress, reason }`.
6. The routing decision passes through the before-tool-call guardrail if the action is destructive. If accepted, `ActionExecuted` is emitted (terminal `ACTIONED`). If rejected, `ActionBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. Independent of the workflow, `ClassificationEvalScorer` (Consumer) listens for `MessageClassified` events, calls `ClassificationJudge`, and writes `ClassificationScored { score, rationale }` back to the entity.
8. The user can click any message card and see the redacted body, the classification reason, the classification score, the routing decision, the guardrail verdict, and the executed action (or blocked action + violations).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InboxSimulator` | `TimedAction` | Drips simulated email messages into `InboxQueue` every 30 s. | scheduler | `InboxQueue` |
| `InboxQueue` | `EventSourcedEntity` | Append-only audit log of every inbound message (`InboundMessageReceived`). | `InboxSimulator`, `InboxEndpoint` | `BodySanitizer` |
| `BodySanitizer` | `Consumer` | Subscribes to `InboxQueue` events; redacts PII; registers `MessageEntity`; starts a `RoutingWorkflow`. | `InboxQueue` events | `MessageEntity`, `RoutingWorkflow` |
| `ClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedMessage` into `URGENT` / `IMPORTANT` / `INFO` / `SPAM` with confidence + reason. | invoked by `RoutingWorkflow` | returns `ClassificationDecision` |
| `RoutingAgent` | `AutonomousAgent` | Owns the `ROUTE` task: decides and returns a typed `RoutingDecision`. | invoked by `RoutingWorkflow` | returns `RoutingDecision` |
| `ActionGuardrail` | `Agent` (typed) | Before-tool-call guardrail: checks any destructive action against a policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `RoutingWorkflow` | returns `GuardrailVerdict` |
| `ClassificationJudge` | `Agent` (typed) | Grades a classification decision against the sanitized message. Returns `ClassificationScore { score 1–5, rationale }`. | invoked by `ClassificationEvalScorer` | returns `ClassificationScore` |
| `RoutingWorkflow` | `Workflow` | Per-message orchestration: sanitize → classify → route → guardrail → execute. | `BodySanitizer` (start) | `MessageEntity` |
| `MessageEntity` | `EventSourcedEntity` | Per-message lifecycle. | `RoutingWorkflow`, `ClassificationEvalScorer` | `MessageView` |
| `MessageView` | `View` | Read-model row per message. | `MessageEntity` events | `InboxEndpoint` |
| `ClassificationEvalScorer` | `Consumer` | Subscribes to `MessageEntity` events; on `MessageClassified` invokes `ClassificationJudge` and writes `ClassificationScored` back. | `MessageEntity` events | `MessageEntity` |
| `InboxEndpoint` | `HttpEndpoint` | `/api/messages/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `MessageView`, `MessageEntity`, `InboxQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and static resources. | — | static resources |

## 5. Data model

```java
record InboundMessage(
    String messageId,
    String senderId,
    String subject,
    String body,
    String channel,            // "gmail" | "imap" | "simulated"
    Instant receivedAt
) {}

record SanitizedMessage(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum MessageLabel { URGENT, IMPORTANT, INFO, SPAM }

record ClassificationDecision(
    MessageLabel label,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum RoutingAction {
    FLAG_URGENT,
    MOVE_TO_FOLDER,
    MARK_READ,
    MOVE_TO_SPAM,
    FORWARD_TO_HUMAN,
    DELETE
}

record RoutingDecision(
    RoutingAction action,
    Optional<String> targetFolder,   // populated when action = MOVE_TO_FOLDER
    Optional<String> forwardAddress, // populated when action = FORWARD_TO_HUMAN
    String reason,
    Instant decidedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record ClassificationScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Message(
    String messageId,
    InboundMessage incoming,
    Optional<SanitizedMessage> sanitized,
    Optional<ClassificationDecision> classification,
    Optional<RoutingDecision> routing,
    Optional<GuardrailVerdict> guardrail,
    Optional<ClassificationScore> classificationScore,
    Optional<String> blockReason,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED,
    SANITIZED,
    CLASSIFIED,
    ROUTING_DECIDED,
    GUARDRAIL_PENDING,
    ACTIONED,
    BLOCKED,
    ESCALATED
}
```

Events on `MessageEntity`: `MessageRegistered`, `MessageSanitized`, `MessageClassified`, `RoutingDecisionRecorded`, `GuardrailVerdictAttached`, `ActionExecuted`, `ActionBlocked`, `MessageEscalated`, `ClassificationScored`.

Events on `InboxQueue`: `InboundMessageReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/messages` — list all messages (newest-first), optional `?label=URGENT|IMPORTANT|INFO|SPAM&status=…` filtered client-side.
- `GET /api/messages/{id}` — one message.
- `POST /api/messages` — manually submit a message (body `InboundMessage` minus `messageId` and `receivedAt`); server assigns both.
- `POST /api/messages/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `ACTIONED` if the operator chooses to execute the blocked action.
- `GET /api/messages/sse` — Server-Sent Events for every message change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: InboxZero Lite — Email Classifier</title>`.

The App UI tab is a three-pane layout: **left** is the message list (status pill + label chip + score chip), **centre** is the selected message's redacted body + classification decision + score, **right** is the routing decision + guardrail verdict + executed action (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `BodySanitizer` Consumer): strips email addresses, phone numbers (E.164 and local formats), national ID patterns, and account number patterns from the message body before any LLM sees it. The categories found are kept for audit; the redacted text is what reaches the agents.
- **G1 — before-tool-call guardrail** on destructive `RoutingAction` values (`MOVE_TO_SPAM`, `DELETE`): checks that the action is consistent with the classification and the sanitized content. Blocking — a violation puts the message in `BLOCKED` for human review.
- **E1 — on-decision eval** (`eval-event`, on the classification decision): `ClassificationEvalScorer` (Consumer) listens for `MessageClassified` events and calls `ClassificationJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate.

## 9. Agent prompts

- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier; returns one of `URGENT`, `IMPORTANT`, `INFO`, `SPAM`; defaults to `INFO` under ambiguity. Never reads the raw body.
- `RoutingAgent` → `prompts/routing-agent.md`. Owns the `ROUTE` task. Returns a `RoutingDecision` with a specific action and reason. Never invents folder names outside the configured list.
- `ActionGuardrail` → `prompts/action-guardrail.md`. Before-tool-call check on destructive actions. Returns `GuardrailVerdict { allowed, violations }`. Conservative — borderline actions are blocked.
- `ClassificationJudge` → `prompts/classification-judge.md`. Grades a classification decision against a 1–5 rubric with one-sentence rationale.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an urgent message → classified `URGENT` → routed to `FLAG_URGENT` → guardrail passes → actioned.
2. **J2** — Simulator drips a spam message → classified `SPAM` → routed to `MOVE_TO_SPAM` → guardrail passes → actioned.
3. **J3** — An ambiguous message classifies as `INFO` and is filed in the general folder; neither destructive action is attempted.
4. **J4** — A message that coaxes a `DELETE` action fails the before-tool-call guardrail; the message lands in `BLOCKED`; the operator can unblock.
5. **J5** — Every classified message carries a `ClassificationScore` (1–5) and rationale within ~10 s of the classification event.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named inbox-classifier demonstrating the handoff-routing × ops-automation cell.
Runs out of the box (in-process simulated inbox; no real Gmail integration required by default).
Maven group io.akka.samples. Maven artifact handoff-routing-ops-automation-inbox-classifier. Java
package io.akka.samples.inboxzeroliteemailclassifier. Akka 3.6.0. HTTP port 9159.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClassifierAgent — classifier. System prompt loaded from
  prompts/classifier-agent.md. Input: SanitizedMessage{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: ClassificationDecision{label: MessageLabel
  (URGENT/IMPORTANT/INFO/SPAM), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to INFO under uncertainty.

- 1 AutonomousAgent RoutingAgent — definition() with capability(TaskAcceptance.of(ROUTE)
  .maxIterationsPerTask(3)). System prompt from prompts/routing-agent.md. Input:
  SanitizedMessage + ClassificationDecision. Output: RoutingDecision{action: RoutingAction,
  targetFolder: Optional<String>, forwardAddress: Optional<String>, reason: String,
  decidedAt: Instant}. Never invents folder names; uses the configured folder list.

- 1 Agent (typed) ActionGuardrail — typed rubric check. System prompt from
  prompts/action-guardrail.md. Input: SanitizedMessage + ClassificationDecision +
  RoutingDecision. Output: GuardrailVerdict{allowed: boolean, violations: List<String>,
  rubricVersion: String}. Only invoked when RoutingDecision.action is MOVE_TO_SPAM or DELETE.
  Used by RoutingWorkflow before executing destructive actions; blocking.

- 1 Agent (typed) ClassificationJudge — judge. System prompt from
  prompts/classification-judge.md. Input: SanitizedMessage + ClassificationDecision. Output:
  ClassificationScore{score: int 1–5, rationale: String, scoredAt: Instant}.

- 1 Workflow RoutingWorkflow per messageId. Steps:
    classifyStep -> routeStep -> guardrailStep (conditional) -> executeStep
  classifyStep calls componentClient.forAgent().inSession(messageId)
    .method(ClassifierAgent::classify).invoke(sanitized). On success emits MessageClassified
    via MessageEntity.recordClassification.
  routeStep calls forAutonomousAgent(RoutingAgent.class, messageId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, classification))) returning
    a taskId, then forTask(taskId).result(InboxTasks.ROUTE) to block on the typed
    RoutingDecision. On success emits RoutingDecisionRecorded.
  guardrailStep is conditional: only executed when routingDecision.action is MOVE_TO_SPAM
    or DELETE. Calls forAgent(...).method(ActionGuardrail::check).invoke(sanitized,
    classification, routingDecision). On verdict.allowed=true proceed to executeStep
    (emits ActionExecuted, terminal ACTIONED). On verdict.allowed=false emit ActionBlocked
    (terminal BLOCKED) and end.
  executeStep for non-destructive actions (FLAG_URGENT, MOVE_TO_FOLDER, MARK_READ,
    FORWARD_TO_HUMAN): emits ActionExecuted directly (no guardrail needed).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on routeStep and executeStep.
    defaultStepRecovery(maxRetries(2).failoverTo(RoutingWorkflow::error)).

- 2 EventSourcedEntities:
    * InboxQueue — append-only audit log. Command receive(InboundMessage) emits
      InboundMessageReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.messageId.
    * MessageEntity (one per messageId) — full per-message lifecycle. State Message{messageId,
      incoming: InboundMessage, Optional<SanitizedMessage> sanitized,
      Optional<ClassificationDecision> classification, Optional<RoutingDecision> routing,
      Optional<GuardrailVerdict> guardrail, Optional<ClassificationScore> classificationScore,
      Optional<String> blockReason, MessageStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. MessageStatus enum: RECEIVED, SANITIZED, CLASSIFIED,
      ROUTING_DECIDED, GUARDRAIL_PENDING, ACTIONED, BLOCKED, ESCALATED.
      Events: MessageRegistered, MessageSanitized, MessageClassified,
      RoutingDecisionRecorded, GuardrailVerdictAttached, ActionExecuted, ActionBlocked,
      MessageEscalated, ClassificationScored. Commands: registerIncoming, attachSanitized,
      recordClassification, recordRoutingDecision, recordGuardrailVerdict, executeAction,
      blockAction, escalate, recordClassificationScore, unblock, getMessage.
      emptyState() returns Message.initial("") with no commandContext() reference.

- 2 Consumers:
    * BodySanitizer subscribed to InboxQueue events; for each InboundMessageReceived
      applies a regex+heuristic redaction pipeline (email addresses, phone numbers in
      E.164 and local formats, national ID patterns matching [A-Z]{2}\d{6,9}, account
      numbers matching ACCT-\w+, plus heuristic name-redaction in greeting positions)
      to subject + body, builds SanitizedMessage with piiCategoriesFound, and calls
      MessageEntity.registerIncoming then attachSanitized for the messageId; then starts
      a RoutingWorkflow with messageId as the workflow id.
    * ClassificationEvalScorer subscribed to MessageEntity events; on MessageClassified
      invokes ClassificationJudge.score(sanitized, decision) and calls
      MessageEntity.recordClassificationScore(messageId, score). On any other event type,
      no-op. Use componentClient — do NOT call the agent from a TimedAction.

- 1 View MessageView with row type MessageRow (mirrors Message; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes MessageEntity events.
  ONE query getAllMessages SELECT * AS messages FROM message_view. No WHERE label or
  WHERE status filter — filter client-side in callers.

- 1 TimedAction InboxSimulator — every 30s, reads next line from
  src/main/resources/sample-events/inbox-messages.jsonl (loops at EOF) and calls
  InboxQueue.receive with a fresh messageId (UUID).

- 2 HttpEndpoints:
    * InboxEndpoint at /api with GET /messages (list from MessageView.getAllMessages,
      filter client-side by ?label and ?status query params), GET /messages/{id},
      POST /messages (body InboundMessage minus messageId/receivedAt — server assigns),
      POST /messages/{id}/unblock (body {decidedBy, note} — operator override:
      executes the blocked action as ACTIONED with an audit note),
      GET /messages/sse (serverSentEventsForView over getAllMessages), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- InboxTasks.java declaring the task constant: ROUTE (resultConformsTo RoutingDecision.class,
  description "Classify and route the email message, returning a typed RoutingDecision").
- Domain records InboundMessage, SanitizedMessage, ClassificationDecision, RoutingDecision,
  GuardrailVerdict, ClassificationScore, and the Message entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9159 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/inbox-messages.jsonl with 9 canned lines (2 URGENT-
  flavoured, 2 IMPORTANT-flavoured, 3 INFO-flavoured, 1 SPAM-flavoured, 1 designed to
  coax a DELETE action to trip the guardrail).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-tool-call. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with purpose.primary_function = email-classification,
  data.data_classes.pii = true, decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false (the system actions messages without HITL by default —
  only blocked destructive actions wait for a human), data.pii_handled_by_sanitizer_before_llm
  = true, failure.failure_modes including "misclassification-urgent-as-spam",
  "destructive-action-on-legitimate-email", "pii-leakage-via-llm",
  "invented-folder-path"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/classifier-agent.md, prompts/routing-agent.md, prompts/action-guardrail.md,
  prompts/classification-judge.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: InboxZero Lite — Email Classifier",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = message list with label chip + status pill + score chip; centre = redacted body
  + classification block; right = routing decision + guardrail verdict + executed action
  or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: InboxZero Lite — Email Classifier</title>. No subtitle on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider block below).
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed to
        the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch. Per-agent mock shapes:
    classifier-agent.json — 12 ClassificationDecision entries: 3 URGENT (keywords like
      "server down", "payment failed", "account locked"), 3 IMPORTANT (meeting requests,
      project updates), 4 INFO (newsletters, confirmations, FYIs), 2 SPAM (prize offers,
      unsolicited marketing). Confidence + one-sentence reason on each.
    routing-agent.json — 9 RoutingDecision entries: 2 FLAG_URGENT (for URGENT messages),
      2 MOVE_TO_FOLDER (for IMPORTANT and INFO), 2 MARK_READ (for INFO), 1 MOVE_TO_SPAM
      (for SPAM), 1 FORWARD_TO_HUMAN (for ambiguous URGENT), 1 DELETE (designed to trip
      the guardrail — a legitimate-looking email incorrectly marked for deletion).
    action-guardrail.json — 10 GuardrailVerdict entries: 7 allowed=true with empty
      violations (for non-destructive and warranted MOVE_TO_SPAM actions), 3 allowed=false
      with one violation each ("unwarranted-delete", "misclassification-contradicts-action",
      "legitimate-sender-marked-spam").
    classification-judge.json — 10 ClassificationScore entries, score 1–5, one-sentence
      rationale covering label-correctness, confidence-calibration, and reason-quality.
- A MockModelProvider.seedFor(messageId) helper makes per-message selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md.
Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded. RoutingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent and declares definition().
- (Lesson 4) Workflow step timeouts overridden: classifyStep 20s, guardrailStep 20s,
  routeStep / executeStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Message is Optional<T>.
- (Lesson 7) InboxTasks.java declares the ROUTE Task<RoutingDecision> constant.
- (Lesson 8) Model names: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build". No "mvn akka:run".
- (Lesson 10) Port 9159 in application.conf; not 9000.
- (Lesson 11) No source.platform string user-facing.
- (Lesson 12) Static UI fits in 1080px content column.
- (Lesson 13) Integration tier label is "Gmail integration (credential required)" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) index.html includes mermaid CSS overrides AND theme variables.
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching by data-tab / data-panel attribute, never NodeList index.
- The BodySanitizer runs INSIDE a Consumer before any LLM call.
- The ClassificationEvalScorer Consumer reacts to MessageClassified events out-of-band.
- The guardrail step happens BEFORE ActionExecuted. A blocked action never appears as
  executed — only as "blocked action + violations" for the operator.
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
