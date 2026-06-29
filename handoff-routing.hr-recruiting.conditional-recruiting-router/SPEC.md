# SPEC — conditional-recruiting-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Conditional Email/Interview Workflow.
**One-line pitch:** An email classifier reads an inbound recruiting email and hands it to an InfoRequester or InterviewOrganizer that owns the outcome end-to-end, with PII redaction before every LLM call and a before-tool-call guardrail on every scheduling and email-send operation.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which path* owns the incoming message, then transfers the full task to a downstream specialist agent. The downstream agent is responsible for the complete outcome; the classifier does not draft or send anything. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw email event and the LLM call. The classifier and the specialists never see raw email addresses, phone numbers, or home addresses.
- A **before-tool-call guardrail** fires before any tool invocation inside `InterviewOrganizer` (the `schedule_slot` and `send_email` tools). It validates that the tool call is well-formed (interviewer id resolves, slot is in the future, candidate consent flag is set) and blocks the call when validation fails, transitioning the application to `TOOL_BLOCKED`.

An **on-decision eval** fires every time the classifier emits a routing decision. A `RoutingJudge` agent grades the decision against the sanitized email on a 1–5 rubric. The score and rationale are written back to the application and visible in the UI.

The pattern is a fan-out-of-one: the workflow branches on the classifier's route, and only the chosen specialist is invoked. The other specialists see no traffic for that application.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live application list. Every application displays its route chip, status pill, eval score, and (if handled) the outbound reply or scheduled slot.
2. `EmailSimulator` (TimedAction) ticks every 30 s and inserts a new canned candidate email from `sample-events/candidate-emails.jsonl` into `InboxQueue`.
3. For each new email: `CandidateSanitizer` (Consumer) redacts the payload, registers an `ApplicationEntity`, and starts a `RecruitingWorkflow`.
4. The workflow calls `EmailClassifierAgent`, gets a `RoutingDecision { route, confidence, reason }`, and emits `RoutingDecided` on the application.
5. Branch on `route`:
   - `INFO_REQUEST` → workflow calls `InfoRequester` with the `REPLY` task and waits for a typed `OutboundReply`.
   - `INTERVIEW_REQUEST` → workflow calls `InterviewOrganizer` with the `SCHEDULE` task. Before each tool call, `ToolCallGuardrail` validates; on failure the workflow emits `ApplicationToolBlocked` (terminal `TOOL_BLOCKED`).
   - `UNROUTABLE` → workflow emits `ApplicationClosed`; ends in `UNROUTABLE_CLOSED`.
6. The specialist returns its result. The workflow emits `ReplyReady` (INFO_REQUEST) or `SlotBooked` (INTERVIEW_REQUEST) and transitions to the terminal `COMPLETED` state.
7. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `RoutingDecided` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the application.
8. The user can click any application card and inspect the redacted email, the routing decision, the eval score, the specialist's reply or calendar hold, and any guardrail block.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EmailSimulator` | `TimedAction` | Drips simulated candidate emails into `InboxQueue` every 30 s. | scheduler | `InboxQueue` |
| `InboxQueue` | `EventSourcedEntity` | Append-only audit log of every inbound email (`InboundEmailReceived`). | `EmailSimulator`, `RecruitingEndpoint` | `CandidateSanitizer` |
| `CandidateSanitizer` | `Consumer` | Subscribes to `InboxQueue` events; redacts PII; registers `ApplicationEntity`; starts a `RecruitingWorkflow`. | `InboxQueue` events | `ApplicationEntity`, `RecruitingWorkflow` |
| `EmailClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedEmail` into `INFO_REQUEST` / `INTERVIEW_REQUEST` / `UNROUTABLE` with confidence + reason. | invoked by `RecruitingWorkflow` | returns `RoutingDecision` |
| `InfoRequester` | `AutonomousAgent` | Owns the `REPLY` task for info-request emails. Returns typed `OutboundReply`. | invoked by `RecruitingWorkflow` | returns `OutboundReply` |
| `InterviewOrganizer` | `AutonomousAgent` | Owns the `SCHEDULE` task for interview-request emails. Uses `availability_lookup` (RAG) and `schedule_slot` (calendar tool). Returns typed `CalendarConfirmation`. | invoked by `RecruitingWorkflow` | returns `CalendarConfirmation` |
| `RoutingJudge` | `Agent` (typed) | Grades a routing decision against the sanitized email. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `ToolCallGuardrail` | `Agent` (typed) | Before-tool-call guardrail: checks a pending tool call (name + args) against the validation rubric. Returns `ToolCallVerdict { allowed, violations }`. | invoked by `RecruitingWorkflow` pre-tool | returns `ToolCallVerdict` |
| `RecruitingWorkflow` | `Workflow` | Per-application orchestration: classify → branch → {reply | schedule} → send. | `CandidateSanitizer` (start) | `ApplicationEntity` |
| `ApplicationEntity` | `EventSourcedEntity` | Per-application lifecycle. | `RecruitingWorkflow`, `RoutingEvalScorer` | `ApplicationView` |
| `ApplicationView` | `View` | Read-model row per application. | `ApplicationEntity` events | `RecruitingEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `ApplicationEntity` events; on `RoutingDecided` invokes `RoutingJudge` and writes `RoutingScored` back. | `ApplicationEntity` events | `ApplicationEntity` |
| `RecruitingEndpoint` | `HttpEndpoint` | `/api/applications/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `ApplicationView`, `ApplicationEntity`, `InboxQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and metadata files. | — | static resources |

## 5. Data model

```java
record InboundEmail(
    String applicationId,
    String candidateId,
    String channel,           // "email" | "careers-portal" | "recruiter-forward"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedEmail(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum ApplicationRoute { INFO_REQUEST, INTERVIEW_REQUEST, UNROUTABLE }

record RoutingDecision(
    ApplicationRoute route,
    String confidence,        // "high" | "medium" | "low"
    String reason             // one short sentence
) {}

enum ReplyAction { QUESTION_ANSWERED, ARTICLE_LINKED, FOLLOW_UP_PROMISED, ESCALATED }

record OutboundReply(
    String replySubject,
    String replyBody,
    ReplyAction action,
    String specialistTag,     // "info-requester"
    Instant repliedAt
) {}

enum ScheduleOutcome { SLOT_BOOKED, CANDIDATE_DEFERRED, ESCALATED_TO_RECRUITER }

record CalendarConfirmation(
    String interviewFormat,   // "phone-screen" | "technical" | "panel" | "executive"
    String interviewerId,
    String proposedSlot,      // ISO-8601 datetime
    ScheduleOutcome outcome,
    String specialistTag,     // "interview-organizer"
    Instant confirmedAt
) {}

record ToolCallVerdict(
    boolean allowed,
    List<String> violations,  // empty when allowed
    String rubricVersion
) {}

record RoutingScore(
    int score,                // 1..5
    String rationale,
    Instant scoredAt
) {}

record Application(
    String applicationId,
    InboundEmail incoming,
    Optional<SanitizedEmail> sanitized,
    Optional<RoutingDecision> routing,
    Optional<OutboundReply> reply,
    Optional<CalendarConfirmation> calendar,
    Optional<ToolCallVerdict> toolGuardrail,
    Optional<RoutingScore> routingScore,
    Optional<String> closureReason,
    ApplicationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ApplicationStatus {
    RECEIVED,
    SANITIZED,
    CLASSIFIED,
    ROUTED_INFO,
    ROUTED_INTERVIEW,
    REPLY_DRAFTED,
    SLOT_PROPOSED,
    TOOL_BLOCKED,
    COMPLETED,
    UNROUTABLE_CLOSED
}
```

Events on `ApplicationEntity`: `ApplicationRegistered`, `ApplicationSanitized`, `RoutingDecided`, `ApplicationRouted`, `ReplyDrafted`, `SlotProposed`, `ToolCallBlocked`, `ApplicationCompleted`, `ApplicationClosed`, `RoutingScored`.

Events on `InboxQueue`: `InboundEmailReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/applications` — list all applications (newest-first), optional `?route=INFO_REQUEST|INTERVIEW_REQUEST|UNROUTABLE&status=…` filtered client-side.
- `GET /api/applications/{id}` — one application.
- `POST /api/applications` — manually submit an email (body `InboundEmail` minus `applicationId` and `receivedAt`); server assigns both.
- `POST /api/applications/{id}/unblock` — body `{ decidedBy, note }` — recruiter override; transitions `TOOL_BLOCKED` to `COMPLETED` if the recruiter approves the blocked tool call.
- `GET /api/applications/sse` — Server-Sent Events for every application change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Conditional Email/Interview Workflow</title>`.

The App UI tab is a three-pane layout: **left** is the application list (status pill + route chip + eval score chip), **centre** is the selected application's redacted email + routing decision + score, **right** is the specialist's output (reply or calendar confirmation) or the guardrail block + Unblock button when `TOOL_BLOCKED`.

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `CandidateSanitizer` Consumer): redacts email addresses, phone numbers, and home addresses from the candidate email before any LLM sees it. The categories found are kept for audit; the redacted text is what reaches the agents.
- **G1 — before-tool-call guardrail** on `InterviewOrganizer`: checks every pending `schedule_slot` or `send_email` tool call against a rubric (interviewer id must resolve, slot must be in the future, candidate consent flag required, no double-booking). Blocking — a violation transitions the application to `TOOL_BLOCKED` for recruiter review.
- **E1 — on-decision eval** (`eval-event`, on the routing decision): `RoutingEvalScorer` (Consumer) listens for `RoutingDecided` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate.

## 9. Agent prompts

- `EmailClassifierAgent` → `prompts/email-classifier-agent.md`. Typed classifier; returns one of `INFO_REQUEST`, `INTERVIEW_REQUEST`, `UNROUTABLE`; defaults to `UNROUTABLE` under ambiguity.
- `InfoRequester` → `prompts/info-requester.md`. Owns the `REPLY` task for info-request emails. Answers from job-posting context only; never invents salary figures.
- `InterviewOrganizer` → `prompts/interview-organizer.md`. Owns the `SCHEDULE` task for interview-request emails. Uses `availability_lookup` and `schedule_slot` tools. Never books a slot in the past.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a routing decision against a 1–5 rubric with one-sentence rationale.
- `ToolCallGuardrail` → `prompts/tool-call-guardrail.md`. Returns a `ToolCallVerdict { allowed, violations }`. Conservative — borderline tool calls are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an info-request email → classified `INFO_REQUEST` → answered by `InfoRequester` → reply sent, status `COMPLETED`.
2. **J2** — Simulator drips an interview-request email → classified `INTERVIEW_REQUEST` → `InterviewOrganizer` queries availability and books a slot → calendar confirmation sent, status `COMPLETED`.
3. **J3** — An unroutable email is classified `UNROUTABLE` and lands in `UNROUTABLE_CLOSED` without any specialist invocation.
4. **J4** — A `schedule_slot` call with a past-dated slot is blocked by the before-tool-call guardrail; the application lands in `TOOL_BLOCKED` and a recruiter can unblock it.
5. **J5** — Every classified application carries a `RoutingScore` (1–5) and rationale within ~10 s of the routing decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named conditional-recruiting-router demonstrating the
handoff-routing × hr-recruiting cell.
Runs out of the box (in-process simulated inbound email stream; no real
email or calendar integration).
Maven group io.akka.samples. Maven artifact
handoff-routing-hr-recruiting-conditional-recruiting-router.
Java package io.akka.samples.conditionalemailinterviewworkflow.
Akka 3.6.0. HTTP port 9529.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) EmailClassifierAgent — classifier.
  System prompt loaded from prompts/email-classifier-agent.md.
  Input: SanitizedEmail{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}.
  Output: RoutingDecision{route: ApplicationRoute
  (INFO_REQUEST/INTERVIEW_REQUEST/UNROUTABLE),
  confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNROUTABLE under uncertainty.

- 1 AutonomousAgent InfoRequester — definition() with
  capability(TaskAcceptance.of(REPLY).maxIterationsPerTask(3)).
  System prompt from prompts/info-requester.md.
  Input: SanitizedEmail + RoutingDecision.
  Output: OutboundReply{replySubject, replyBody, action: ReplyAction,
  specialistTag = "info-requester", repliedAt}.
  Never invents salary figures or equity terms; sets action=ESCALATED
  when asked for undisclosed information.

- 1 AutonomousAgent InterviewOrganizer — definition() with
  capability(TaskAcceptance.of(SCHEDULE).maxIterationsPerTask(5)).
  System prompt from prompts/interview-organizer.md. Same input shape;
  specialistTag = "interview-organizer". Uses tools availability_lookup
  (RAG over simulated interviewer calendar blocks) and schedule_slot
  (writes a CalendarConfirmation). Returns CalendarConfirmation.
  Every tool call is guarded by a ToolCallGuardrail check inside the
  workflow's toolGuardrailStep.

- 1 Agent (typed) RoutingJudge — judge. System prompt from
  prompts/routing-judge.md.
  Input: SanitizedEmail + RoutingDecision.
  Output: RoutingScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Agent (typed) ToolCallGuardrail — typed rubric check.
  System prompt from prompts/tool-call-guardrail.md.
  Input: pendingTool: String (tool name), toolArgs: Map<String,Object>.
  Output: ToolCallVerdict{allowed: boolean, violations: List<String>,
  rubricVersion: String}. Used by RecruitingWorkflow before any tool
  invocation; blocking.

- 1 Workflow RecruitingWorkflow per applicationId. Steps:
    classifyStep -> routeStep ->
      {infoStep | scheduleStep | closeStep}
    classifyStep calls
      componentClient.forAgent().inSession(applicationId)
        .method(EmailClassifierAgent::classify).invoke(sanitized).
      On success emits RoutingDecided via ApplicationEntity.recordRouting.
    routeStep branches on RoutingDecision.route:
      INFO_REQUEST -> proceed to infoStep
        (emits ApplicationRouted{INFO_REQUEST})
      INTERVIEW_REQUEST -> proceed to scheduleStep
        (emits ApplicationRouted{INTERVIEW_REQUEST})
      UNROUTABLE -> closeStep
        (emits ApplicationClosed; terminates UNROUTABLE_CLOSED).
    infoStep calls forAutonomousAgent(InfoRequester.class, applicationId)
      .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, routing)))
      returning a taskId, then forTask(taskId).result(RecruitingTasks.REPLY)
      to block on the typed OutboundReply. On success emits ReplyDrafted
      then ApplicationCompleted (terminal COMPLETED).
    scheduleStep calls forAutonomousAgent(
      InterviewOrganizer.class, applicationId)
      .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, routing)))
      returning a taskId, then forTask(taskId).result(
      RecruitingTasks.SCHEDULE) to block on CalendarConfirmation.
      Before each tool call in the agent loop, toolGuardrailStep validates
      via ToolCallGuardrail.check(toolName, toolArgs). On verdict.allowed=
      false emit ToolCallBlocked (terminal TOOL_BLOCKED) and end.
      On success emits SlotProposed then ApplicationCompleted (terminal
      COMPLETED).
    Override settings() with stepTimeout(Duration.ofSeconds(20)) on
      classifyStep and toolGuardrailStep, stepTimeout(Duration.ofSeconds(
      60)) on infoStep, scheduleStep.
      defaultStepRecovery(maxRetries(2).failoverTo(
      RecruitingWorkflow::error)).

- 2 EventSourcedEntities:
    * InboxQueue — append-only audit log. Command receive(InboundEmail)
      emits InboundEmailReceived{incoming}. Idempotent on
      incoming.applicationId.
    * ApplicationEntity (one per applicationId) — full per-application
      lifecycle. State Application{applicationId, incoming: InboundEmail,
      Optional<SanitizedEmail> sanitized, Optional<RoutingDecision>
      routing, Optional<OutboundReply> reply,
      Optional<CalendarConfirmation> calendar,
      Optional<ToolCallVerdict> toolGuardrail,
      Optional<RoutingScore> routingScore,
      Optional<String> closureReason,
      ApplicationStatus status, Instant createdAt,
      Optional<Instant> finishedAt}.
      ApplicationStatus enum: RECEIVED, SANITIZED, CLASSIFIED,
      ROUTED_INFO, ROUTED_INTERVIEW, REPLY_DRAFTED, SLOT_PROPOSED,
      TOOL_BLOCKED, COMPLETED, UNROUTABLE_CLOSED.
      Events: ApplicationRegistered, ApplicationSanitized,
      RoutingDecided, ApplicationRouted, ReplyDrafted, SlotProposed,
      ToolCallBlocked, ApplicationCompleted, ApplicationClosed,
      RoutingScored.
      Commands: registerIncoming, attachSanitized, recordRouting,
      recordApplicationRouted, recordReplyDrafted, recordSlotProposed,
      recordToolCallBlocked, complete, close, recordRoutingScore,
      unblock, getApplication. emptyState() returns
      Application.initial("") with no commandContext() reference.

- 2 Consumers:
    * CandidateSanitizer subscribed to InboxQueue events; for each
      InboundEmailReceived applies a regex+heuristic redaction pipeline
      (email addresses, phone numbers, home addresses via postal-code
      pattern) to subject + body, builds SanitizedEmail with
      piiCategoriesFound, calls ApplicationEntity.registerIncoming then
      attachSanitized, then starts a RecruitingWorkflow with applicationId
      as the workflow id.
    * RoutingEvalScorer subscribed to ApplicationEntity events; on
      RoutingDecided invokes RoutingJudge.score(sanitized, decision) and
      calls ApplicationEntity.recordRoutingScore(applicationId, score).
      On any other event type, no-op. Use componentClient — do NOT
      call the agent from a TimedAction.

- 1 View ApplicationView with row type ApplicationRow (mirrors Application;
  uses Optional<T> for every nullable lifecycle field per Lesson 6).
  Table updater consumes ApplicationEntity events.
  ONE query getAllApplications SELECT * AS applications FROM
  application_view. No WHERE route or WHERE status filter — filter
  client-side in callers.

- 1 TimedAction EmailSimulator — every 30s, reads next line from
  src/main/resources/sample-events/candidate-emails.jsonl (loops at EOF)
  and calls InboxQueue.receive with a fresh applicationId (UUID).

- 2 HttpEndpoints:
    * RecruitingEndpoint at /api with GET /applications (list from
      ApplicationView.getAllApplications, filter client-side by ?route
      and ?status query params), GET /applications/{id},
      POST /applications (body InboundEmail minus applicationId/receivedAt
      — server assigns), POST /applications/{id}/unblock (body
      {decidedBy, note} — recruiter override: completes the blocked
      application), GET /applications/sse (serverSentEventsForView
      over getAllApplications), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving
      the YAML/MD files from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
      static-resources/*.

Companion files:
- RecruitingTasks.java declaring task constants: REPLY (resultConformsTo
  OutboundReply.class, description "Answer the candidate's question and
  return a typed OutboundReply") and SCHEDULE (resultConformsTo
  CalendarConfirmation.class, description "Book the interview and return
  a typed CalendarConfirmation").
- Domain records InboundEmail, SanitizedEmail, RoutingDecision,
  OutboundReply, CalendarConfirmation, ToolCallVerdict, RoutingScore,
  and the Application entity state.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9529 and the three model-provider
  blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/candidate-emails.jsonl with 9 canned
  lines (3 INFO_REQUEST-flavoured, 3 INTERVIEW_REQUEST-flavoured, 2
  UNROUTABLE-flavoured, 1 designed to trigger the guardrail with a
  past-dated slot request).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the root-level files for the endpoint to serve
  from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer
  pii, G1 guardrail before-tool-call, and E1 eval-event on-decision-eval.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with purpose.primary_function =
  hr-recruiting, data.data_classes.pii = true,
  decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false, failure.failure_modes including
  "wrong-route-classification", "inappropriate-reply-content",
  "pii-leakage-via-llm", "double-booking", "past-dated-slot";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/email-classifier-agent.md, prompts/info-requester.md,
  prompts/interview-organizer.md, prompts/routing-judge.md,
  prompts/tool-call-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Conditional
  Email/Interview Workflow", prerequisites, generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained
  file (no ui/, no npm). Five tabs matching the formal exemplar. App UI
  tab uses a three-column layout (left = application list with route
  chip + status pill + eval score chip; centre = redacted email + routing
  decision block; right = specialist output + guardrail verdict or
  violations + Unblock button when TOOL_BLOCKED). Browser title exactly:
  <title>Akka Sample: Conditional Email/Interview Workflow</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment
  for ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY.
  If exactly one is set, default application.conf's model-provider to
  match and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options:
    (a) Mock LLM — no real key; generate a MockModelProvider that
        returns random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the
        Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a
        project-local .akka-build.yaml; /akka:build sources the file
        before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at
        run time.
    (e) Type once in this session — value lives in Claude session
        memory; passed to the JVM via the MCP tool's environment
        parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming
  the configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider
  interface with a per-agent dispatch on the agent class or Task<R> id.
  Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by applicationId).
- Per-agent mock-response shapes for THIS blueprint:
    email-classifier-agent.json — 12 RoutingDecision entries spanning
      INFO_REQUEST (salary questions, start-date questions, benefits
      questions), INTERVIEW_REQUEST (scheduling asks, availability
      submissions, reschedule requests), and UNROUTABLE (spam,
      off-topic, one-word messages). Confidence + a one-sentence
      reason on each.
    info-requester.json — 8 OutboundReply entries: 5 with action
      QUESTION_ANSWERED or ARTICLE_LINKED (salary policy, benefits
      overview, remote-work policy), 1 with FOLLOW_UP_PROMISED, 1 with
      ESCALATED (undisclosed equity ask), 1 designed to trip an
      invented-salary-figure guardrail (if one were added).
    interview-organizer.json — 8 CalendarConfirmation entries: 5
      SLOT_BOOKED across phone-screen / technical / panel formats,
      1 CANDIDATE_DEFERRED (no slots available this week), 1
      ESCALATED_TO_RECRUITER, 1 with a past-dated proposedSlot
      designed to trip the before-tool-call guardrail.
    routing-judge.json — 10 RoutingScore entries, score 1–5,
      one-sentence rationale.
    tool-call-guardrail.json — 10 ToolCallVerdict entries. 7 with
      allowed=true. 3 with allowed=false and violations: one each of
      "past-dated-slot", "interviewer-id-not-found",
      "consent-flag-missing".

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  InfoRequester and InterviewOrganizer both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare
  definition().
- (Lesson 4) Workflow step timeouts overridden via settings():
  classifyStep 20s, toolGuardrailStep 20s, infoStep /
  scheduleStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Application is
  Optional<T>. The ApplicationView row type uses the same wrapping.
- (Lesson 7) RecruitingTasks.java declares the REPLY Task<OutboundReply>
  and SCHEDULE Task<CalendarConfirmation> constants. Both specialists'
  definition().capability(TaskAcceptance.of(...)) reference them.
- (Lesson 8) Model names verified against current lineup:
  claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere.
- (Lesson 10) Port 9529 in application.conf; not 9000.
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
- The CandidateSanitizer runs INSIDE a Consumer before any LLM call —
  not inside an Agent's prompt and not after the LLM has seen the raw
  payload.
- The RoutingEvalScorer Consumer reacts to RoutingDecided events and
  calls RoutingJudge via componentClient.forAgent(). It does NOT modify
  the workflow flow — the eval is out-of-band metadata.
- The before-tool-call guardrail fires BEFORE the tool executes. A
  blocked tool call never fires the underlying calendar or email API.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
