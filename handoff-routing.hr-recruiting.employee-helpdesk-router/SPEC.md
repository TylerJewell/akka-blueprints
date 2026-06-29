# SPEC — employee-helpdesk-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Employee Helpdesk Router.
**One-line pitch:** A topic-routing agent classifies an incoming employee question into HR, IT, or policy and hands the `ANSWER` task off to the right specialist end-to-end, with PII redaction before any LLM call and a before-response guardrail that checks every draft answer against the internal-policy rubric.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides which specialist should own the answer, then transfers the same task identity to that specialist, which produces the final response. The classifier does not narrate or summarise. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw question event and the LLM call. The router and the specialists never see raw employee ids, email addresses, or phone numbers. Employee identifiers are particularly sensitive because they can link a question about, say, a leave request or a performance improvement plan back to a specific person by name.
- A **before-agent-response guardrail** runs on the specialist's draft answer before it is published. It checks the draft against an internal-policy rubric: no fabricated policy document references, no citation of specific benefit amounts that were not stated in the question, no legal or medical advice, no echoing of `[REDACTED]` tokens. Blocking — a rubric violation keeps the answer in `BLOCKED` for HR-team review; the employee does not see the blocked draft.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live question list. Every question displays its topic chip, status pill, route score, and (if answered) the published response.
2. `QuestionSimulator` (TimedAction) ticks every 30 s and inserts a new canned employee question from `sample-events/employee-questions.jsonl` into `QuestionQueue`.
3. For each new question: `PiiSanitizer` (Consumer) redacts the payload, registers a `QuestionEntity`, and starts a `HelpWorkflow`.
4. The workflow calls `TopicRouter`, gets a `RoutingDecision { topic, confidence, reason }`, and emits `QuestionRouted` on the entity.
5. Branch on `topic`:
   - `HR` → workflow calls `HRSpecialist` with the `ANSWER` task and waits for a typed `Answer` result.
   - `IT` → workflow calls `ITSpecialist` with the same `ANSWER` task.
   - `POLICY` → workflow calls `PolicySpecialist` with the same `ANSWER` task.
   - `UNCLEAR` → workflow emits `QuestionEscalated`; ends.
6. The specialist's draft `Answer` passes through `AnswerGuardrail`. If accepted, `AnswerPublished` is emitted (terminal `ANSWERED`). If rejected, `AnswerBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. Independent of the workflow, `RouteEvalScorer` (Consumer) listens for `QuestionRouted` events, calls `RouteJudge`, and writes `RouteScored { score, rationale }` back to the entity.
8. The user can click any question card and see the redacted question, the routing reason, the route score, the chosen specialist, the draft (or blocked draft + violations), and the published answer.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QuestionSimulator` | `TimedAction` | Drips simulated employee questions into `QuestionQueue` every 30 s. | scheduler | `QuestionQueue` |
| `QuestionQueue` | `EventSourcedEntity` | Append-only audit log of every inbound question (`InboundQuestionReceived`). | `QuestionSimulator`, `HelpdeskEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `QuestionQueue` events; redacts PII; registers `QuestionEntity`; starts a `HelpWorkflow`. | `QuestionQueue` events | `QuestionEntity`, `HelpWorkflow` |
| `TopicRouter` | `Agent` (typed, not autonomous) | Classifies a `SanitizedQuestion` into `HR` / `IT` / `POLICY` / `UNCLEAR` with confidence + reason. | invoked by `HelpWorkflow` | returns `RoutingDecision` |
| `HRSpecialist` | `AutonomousAgent` | Owns the `ANSWER` task for HR questions. Returns typed `Answer`. | invoked by `HelpWorkflow` | returns `Answer` |
| `ITSpecialist` | `AutonomousAgent` | Owns the `ANSWER` task for IT questions. Returns typed `Answer`. | invoked by `HelpWorkflow` | returns `Answer` |
| `PolicySpecialist` | `AutonomousAgent` | Owns the `ANSWER` task for policy questions. Returns typed `Answer`. | invoked by `HelpWorkflow` | returns `Answer` |
| `RouteJudge` | `Agent` (typed) | Grades a routing decision against the sanitized question. Returns `RouteScore { score 1–5, rationale }`. | invoked by `RouteEvalScorer` | returns `RouteScore` |
| `AnswerGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `Answer` against the policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `HelpWorkflow` | returns `GuardrailVerdict` |
| `HelpWorkflow` | `Workflow` | Per-question orchestration: route → branch → answer → guardrail → publish. | `PiiSanitizer` (start) | `QuestionEntity` |
| `QuestionEntity` | `EventSourcedEntity` | Per-question lifecycle. | `HelpWorkflow`, `RouteEvalScorer` | `QuestionView` |
| `QuestionView` | `View` | Read-model row per question. | `QuestionEntity` events | `HelpdeskEndpoint` |
| `RouteEvalScorer` | `Consumer` | Subscribes to `QuestionEntity` events; on `QuestionRouted` invokes `RouteJudge` and writes `RouteScored` back. | `QuestionEntity` events | `QuestionEntity` |
| `HelpdeskEndpoint` | `HttpEndpoint` | `/api/questions/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `QuestionView`, `QuestionEntity`, `QuestionQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingQuestion(
    String questionId,
    String employeeId,
    String channel,              // "portal" | "chat" | "email"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedQuestion(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum QuestionTopic { HR, IT, POLICY, UNCLEAR }

record RoutingDecision(
    QuestionTopic topic,
    String confidence,           // "high" | "medium" | "low"
    String reason                // one short sentence
) {}

enum AnswerAction { INFO_PROVIDED, POLICY_CITED, TICKET_OPENED, FOLLOW_UP_SCHEDULED, ESCALATED }

record Answer(
    String responseSubject,
    String responseBody,
    AnswerAction action,
    String specialistTag,        // "hr" | "it" | "policy"
    Instant answeredAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,     // empty when allowed
    String rubricVersion
) {}

record RouteScore(
    int score,                   // 1..5
    String rationale,
    Instant scoredAt
) {}

record Question(
    String questionId,
    IncomingQuestion incoming,
    Optional<SanitizedQuestion> sanitized,
    Optional<RoutingDecision> routing,
    Optional<Answer> answer,
    Optional<GuardrailVerdict> guardrail,
    Optional<RouteScore> routeScore,
    Optional<String> escalationReason,
    QuestionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QuestionStatus {
    RECEIVED,
    SANITIZED,
    ROUTED,
    ROUTED_HR,
    ROUTED_IT,
    ROUTED_POLICY,
    ANSWER_DRAFTED,
    BLOCKED,
    ANSWERED,
    ESCALATED
}
```

Events on `QuestionEntity`: `QuestionRegistered`, `QuestionSanitized`, `QuestionRouted`, `RoutingBranched`, `AnswerDrafted`, `GuardrailVerdictAttached`, `AnswerPublished`, `AnswerBlocked`, `QuestionEscalated`, `RouteScored`.

Events on `QuestionQueue`: `InboundQuestionReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/questions` — list all questions (newest-first), optional `?topic=HR|IT|POLICY|UNCLEAR&status=…` filtered client-side.
- `GET /api/questions/{id}` — one question.
- `POST /api/questions` — manually submit a question (body `IncomingQuestion` minus `questionId` and `receivedAt`); server assigns both.
- `POST /api/questions/{id}/unblock` — body `{ decidedBy, note }` — HR-team override; transitions `BLOCKED` to `ANSWERED` if the reviewer chooses to publish the blocked draft.
- `GET /api/questions/sse` — Server-Sent Events for every question change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Employee Helpdesk Router</title>`.

The App UI tab is a three-pane layout: **left** is the question list (status pill + topic chip + score chip), **centre** is the selected question's redacted subject + body + routing block + score block, **right** is the chosen specialist's draft + guardrail verdict + published answer (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts employee ids (`EMP-\d+`), email addresses, and phone numbers from the question subject and body before any LLM sees it. The categories found are kept for audit; only the redacted text reaches the agents.
- **G1 — before-agent-response guardrail** on `HRSpecialist`, `ITSpecialist`, and `PolicySpecialist`: checks every draft `Answer` against a rubric (no fabricated policy document ids, no specific benefit-amount claims that were not in the question, no legal/medical advice, no echoing of `[REDACTED]` tokens). Blocking — a violation puts the question in `BLOCKED` for HR-team review.

## 9. Agent prompts

- `TopicRouter` → `prompts/topic-router.md`. Typed classifier; returns one of `HR`, `IT`, `POLICY`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `HRSpecialist` → `prompts/hr-specialist.md`. Owns the `ANSWER` task for HR questions. Never invents policy document ids or specific benefit amounts.
- `ITSpecialist` → `prompts/it-specialist.md`. Owns the `ANSWER` task for IT questions. Never invents ticket ids or version numbers.
- `PolicySpecialist` → `prompts/policy-specialist.md`. Owns the `ANSWER` task for policy questions. Cites only policy documents that appeared in the question context; never invents one.
- `RouteJudge` → `prompts/route-judge.md`. Grades a routing decision against a 1–5 rubric with one-sentence rationale.
- `AnswerGuardrail` → `prompts/answer-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an HR-flavoured question → routed `HR` → answered by `HRSpecialist` → guardrail passes → published.
2. **J2** — Simulator drips an IT question → routed `IT` → answered by `ITSpecialist` → guardrail passes → published.
3. **J3** — Simulator drips a policy question → routed `POLICY` → answered by `PolicySpecialist` → guardrail passes → published.
4. **J4** — An ambiguous question routes to `UNCLEAR` and lands in `ESCALATED` without invoking any specialist.
5. **J5** — A draft that cites a fabricated policy document id is blocked; the question lands in `BLOCKED` with the violation listed.
6. **J6** — Every routed question carries a `RouteScore` (1–5) within ~10 s of the routing decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named employee-helpdesk-router demonstrating the
handoff-routing × hr-recruiting cell.
Runs out of the box (in-process simulated inbound stream; no real HR system integration).
Maven group io.akka.samples. Maven artifact
handoff-routing-hr-recruiting-employee-helpdesk-router. Java package
io.akka.samples.employeeagent. Akka 3.6.0. HTTP port 9521.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) TopicRouter — classifier. System prompt loaded from
  prompts/topic-router.md. Input: SanitizedQuestion{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: RoutingDecision{topic: QuestionTopic
  (HR/IT/POLICY/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent HRSpecialist — definition() with capability(TaskAcceptance.of(ANSWER)
  .maxIterationsPerTask(3)). System prompt from prompts/hr-specialist.md. Input:
  SanitizedQuestion + RoutingDecision. Output: Answer{responseSubject, responseBody,
  action: AnswerAction, specialistTag = "hr", answeredAt}. Never invents policy document
  ids or specific benefit amounts; sets action=ESCALATED with a reason when outside
  authority.

- 1 AutonomousAgent ITSpecialist — definition() with capability(TaskAcceptance.of(ANSWER)
  .maxIterationsPerTask(3)). System prompt from prompts/it-specialist.md. Same input
  shape; specialistTag = "it". Never invents ticket ids or version numbers; sets
  action=TICKET_OPENED when the issue requires hands-on investigation.

- 1 AutonomousAgent PolicySpecialist — definition() with capability(TaskAcceptance.of(ANSWER)
  .maxIterationsPerTask(3)). System prompt from prompts/policy-specialist.md. Same input
  shape; specialistTag = "policy". Cites only policy documents named in the question;
  never invents a document id.

- 1 Agent (typed) RouteJudge — judge. System prompt from prompts/route-judge.md. Input:
  SanitizedQuestion + RoutingDecision. Output: RouteScore{score: int 1–5, rationale:
  String, scoredAt: Instant}.

- 1 Agent (typed) AnswerGuardrail — typed rubric check. System prompt from
  prompts/answer-guardrail.md. Input: SanitizedQuestion + Answer. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by HelpWorkflow before publishing; blocking.

- 1 Workflow HelpWorkflow per questionId. Steps:
    routeStep -> branchStep -> {hrStep | itStep | policyStep | escalateStep}
               -> guardrailStep -> publishStep
  routeStep calls componentClient.forAgent().inSession(questionId)
    .method(TopicRouter::route).invoke(sanitized). On success emits QuestionRouted via
    QuestionEntity.recordRouting.
  branchStep branches on RoutingDecision.topic:
    HR -> proceed to hrStep (emits RoutingBranched{HR})
    IT -> proceed to itStep (emits RoutingBranched{IT})
    POLICY -> proceed to policyStep (emits RoutingBranched{POLICY})
    UNCLEAR -> escalateStep (emits QuestionEscalated; terminates).
  hrStep / itStep / policyStep call forAutonomousAgent(<Specialist>.class, questionId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, routing))) returning a
    taskId, then forTask(taskId).result(HelpTasks.ANSWER) to block on the typed Answer.
    On success emits AnswerDrafted.
  guardrailStep calls forAgent(...).method(AnswerGuardrail::check).invoke(sanitized,
    draft). On verdict.allowed=true proceed to publishStep (emits AnswerPublished,
    terminal ANSWERED). On verdict.allowed=false emit AnswerBlocked (terminal BLOCKED)
    and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on routeStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on hrStep, itStep, policyStep,
    and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(HelpWorkflow::error)).

- 2 EventSourcedEntities:
    * QuestionQueue — append-only audit log. Command receive(IncomingQuestion) emits
      InboundQuestionReceived{incoming}. No mutable state beyond a counter; commands
      are idempotent on incoming.questionId.
    * QuestionEntity (one per questionId) — full per-question lifecycle. State
      Question{questionId, incoming: IncomingQuestion, Optional<SanitizedQuestion>
      sanitized, Optional<RoutingDecision> routing, Optional<Answer> answer,
      Optional<GuardrailVerdict> guardrail, Optional<RouteScore> routeScore,
      Optional<String> escalationReason, QuestionStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. QuestionStatus enum: RECEIVED, SANITIZED,
      ROUTED, ROUTED_HR, ROUTED_IT, ROUTED_POLICY, ANSWER_DRAFTED, BLOCKED,
      ANSWERED, ESCALATED. Events: QuestionRegistered, QuestionSanitized,
      QuestionRouted, RoutingBranched, AnswerDrafted, GuardrailVerdictAttached,
      AnswerPublished, AnswerBlocked, QuestionEscalated, RouteScored. Commands:
      registerIncoming, attachSanitized, recordRouting, recordBranch, recordDraft,
      recordGuardrailVerdict, publish, block, escalate, recordRouteScore, unblock,
      getQuestion. emptyState() returns Question.initial("") with no commandContext()
      reference.

- 2 Consumers:
    * PiiSanitizer subscribed to QuestionQueue events; for each InboundQuestionReceived
      applies a regex+heuristic redaction pipeline (emails, phone numbers,
      EMP-\d+ employee ids) to subject + body, builds SanitizedQuestion with
      piiCategoriesFound, and calls QuestionEntity.registerIncoming then
      attachSanitized for the questionId; then starts a HelpWorkflow with questionId
      as the workflow id.
    * RouteEvalScorer subscribed to QuestionEntity events; on QuestionRouted invokes
      RouteJudge.score(sanitized, decision) and calls QuestionEntity.recordRouteScore(
      questionId, score). On any other event type, no-op. Use componentClient — do
      NOT call the agent from a TimedAction.

- 1 View QuestionView with row type QuestionRow (mirrors Question; uses Optional<T>
  for every nullable lifecycle field per Lesson 6). Table updater consumes
  QuestionEntity events. ONE query getAllQuestions SELECT * AS questions FROM
  question_view. No WHERE topic or WHERE status filter (Akka cannot auto-index enum
  columns) — filter client-side in callers.

- 1 TimedAction QuestionSimulator — every 30s, reads next line from
  src/main/resources/sample-events/employee-questions.jsonl (loops at EOF) and calls
  QuestionQueue.receive with a fresh questionId (UUID).

- 2 HttpEndpoints:
    * HelpdeskEndpoint at /api with GET /questions (list from QuestionView.getAllQuestions,
      filter client-side by ?topic and ?status query params), GET /questions/{id},
      POST /questions (body IncomingQuestion minus questionId/receivedAt — server
      assigns), POST /questions/{id}/unblock (body {decidedBy, note} — HR-team
      override: publishes the blocked draft as ANSWERED with an audit note),
      GET /questions/sse (serverSentEventsForView over getAllQuestions), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD
      files from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- HelpTasks.java declaring the task constants: ANSWER (resultConformsTo Answer.class,
  description "Answer the employee question end-to-end and return a typed Answer").
- Domain records IncomingQuestion, SanitizedQuestion, RoutingDecision, Answer,
  GuardrailVerdict, RouteScore, and the Question entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9521 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/employee-questions.jsonl with 10 canned lines (3
  HR-flavoured, 3 IT-flavoured, 2 POLICY-flavoured, 1 UNCLEAR-flavoured, 1 designed
  to trip the guardrail with a fabricated policy document id).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-agent-response policy-rubric. Matching simplified_view list. No
  regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = employee-helpdesk,
  data.data_classes.pii = true, decisions.authority_level = draft-only,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  blocked drafts wait for a human), data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-topic-routing", "fabricated-policy-reference",
  "pii-leakage-via-llm", "benefit-amount-fabrication"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/topic-router.md, prompts/hr-specialist.md, prompts/it-specialist.md,
  prompts/policy-specialist.md, prompts/route-judge.md, prompts/answer-guardrail.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Employee Helpdesk Router",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column
  layout (left = question list with topic chip + status pill + score chip; centre =
  redacted question + routing block; right = specialist draft + guardrail verdict +
  published answer or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Employee Helpdesk Router</title>. No subtitle on the Overview
  tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch. Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry pseudo-randomly
  per call (seeded by questionId so reruns are deterministic).
- Per-agent mock-response shapes for THIS blueprint:
    topic-router.json — 12 RoutingDecision entries spanning HR (leave requests,
      payroll questions, onboarding queries), IT (password resets, software access,
      device requests), POLICY (code-of-conduct questions, data-handling procedures,
      travel-expense rules), and UNCLEAR (mixed content, very short messages).
    hr-specialist.json — 8 Answer entries: 5 with action INFO_PROVIDED or
      FOLLOW_UP_SCHEDULED (well within authority), 1 with ESCALATED (benefit amount
      outside authority), 1 with POLICY_CITED, 1 designed to trip the guardrail
      (cites a fabricated policy document id "HR-POL-9991") so guardrail tests have
      material.
    it-specialist.json — 8 Answer entries: 5 with INFO_PROVIDED or TICKET_OPENED,
      1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED, 1 designed to trip the guardrail
      (echoes a [REDACTED] token in the response body).
    policy-specialist.json — 8 Answer entries: 5 with POLICY_CITED or INFO_PROVIDED,
      1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED, 1 designed to trip the guardrail
      (gives specific legal advice about a regulatory compliance question).
    route-judge.json — 10 RouteScore entries, score 1–5, one-sentence rationale.
    answer-guardrail.json — 10 GuardrailVerdict entries. 7 with allowed=true and
      empty violations. 3 with allowed=false: "invented-policy-reference",
      "echoes-redacted-token", "out-of-scope-legal-advice".

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent. HRSpecialist,
  ITSpecialist, and PolicySpecialist all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): routeStep 20s,
  guardrailStep 20s, hrStep / itStep / policyStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Question is Optional<T>. The
  QuestionView row type uses the same Optional wrapping.
- (Lesson 7) HelpTasks.java declares the ANSWER Task<Answer> constant. All three
  specialists' definition().capability(TaskAcceptance.of(ANSWER)...) reference it.
- (Lesson 8) Model names verified: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9521 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides AND
  theme variables (state-diagram label colour white, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. No "hidden" zombie panels.
- The PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside an
  Agent's prompt and not after the LLM has seen the raw payload.
- The RouteEvalScorer Consumer reacts to QuestionRouted events and calls RouteJudge
  via componentClient.forAgent(). It does NOT modify the workflow flow.
- The guardrail step happens BEFORE AnswerPublished. A blocked draft never reaches
  the UI as published.
- No forbidden words in user-facing text.
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
