# SPEC — llm-auditor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** LLM Auditor.
**One-line pitch:** Submit a chatbot response; a critic agent audits it against accuracy, tone, and policy rules; a reviser agent rewrites flagged responses; the two iterate until the critic approves or the revision ceiling is hit.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to governance risk. A Workflow alternates between a reviewer agent (`CriticAgent`) and a rewriter agent (`ReviserAgent`), feeding each audit's findings back into the next revision until the critic passes the output or a halt ends the loop. The blueprint also demonstrates two governance mechanisms: an **eval-event** that records every cycle's verdict for downstream quality measurement and compliance reporting, and an **output guardrail** that intercepts responses whose initial severity score exceeds a blocking threshold before the revision loop even starts — sending them directly to `ESCALATED` so a human operator can review.

## 3. User-facing flows

The user opens the App UI tab and submits a chatbot response (the raw response text plus the session channel it came from).

1. The system creates a `Session` record in `AUDITING` and starts an `AuditWorkflow`.
2. The guardrail runs first: it checks the response's initial severity against a configured blocking threshold. Responses above the threshold are immediately escalated with `reasonCode = SEVERITY_EXCEEDS_THRESHOLD`; they never enter the revision loop.
3. For responses that pass the guardrail, the Critic evaluates the response against a four-dimension rubric (accuracy, tone, policy compliance, hallucination risk) and returns either `PASS` with a one-line rationale, or `REVISE` with a typed `AuditFindings` payload (up to four findings).
4. On `PASS`, the workflow transitions the session to `APPROVED` with the passing response text and the critic's rationale.
5. On `REVISE`, the workflow records the cycle, the audit findings, and the critic's verdict on the entity, then calls the Reviser with the findings attached. The Reviser produces a revised response.
6. The revised response re-enters the critic step. The loop continues until `PASS` or the ceiling.
7. If the loop reaches `maxRevisions` (default 3) without a `PASS`, the halt activates: the workflow ends with `ESCALATED`, the lowest-severity revision is preserved on the entity along with every finding set for the compliance record, and an `AuditRecorded` event captures the escalation.

A `ResponseSimulator` (TimedAction) drips a canned chatbot response every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CriticAgent` | `AutonomousAgent` | Audits a chatbot response against the rubric; returns `PASS` or `REVISE` with findings. | `AuditWorkflow` | returns `AuditVerdict` to workflow |
| `ReviserAgent` | `AutonomousAgent` | Rewrites a flagged response, addressing every finding before returning. | `AuditWorkflow` | returns `RevisedResponse` to workflow |
| `AuditWorkflow` | `Workflow` | Runs the guardrail → audit → revise loop; halts at the ceiling. | `AuditEndpoint`, `ResponseIngestConsumer` | `SessionEntity` |
| `SessionEntity` | `EventSourcedEntity` | Holds the audit lifecycle, every revision, every finding set, and the final disposition. | `AuditWorkflow` | `SessionsView` |
| `ResponseQueue` | `EventSourcedEntity` | Logs each inbound response for replay and compliance record-keeping. | `AuditEndpoint`, `ResponseSimulator` | `ResponseIngestConsumer` |
| `SessionsView` | `View` | List-of-sessions read model. | `SessionEntity` events | `AuditEndpoint` |
| `ResponseIngestConsumer` | `Consumer` | Subscribes to `ResponseQueue` events; starts a workflow per inbound response. | `ResponseQueue` events | `AuditWorkflow` |
| `ResponseSimulator` | `TimedAction` | Drips a sample response every 60 s from `sample-events/chatbot-responses.jsonl`. | scheduler | `ResponseQueue` |
| `AuditSampler` | `TimedAction` | Every 30 s, scans `SessionsView`, records an `AuditRecorded` event for any cycle completed since the last tick. | scheduler | `SessionEntity` |
| `AuditEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SessionsView`, `ResponseQueue`, `SessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ChatbotResponse(String text, String channel, Instant receivedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail, int initialSeverity) {}

record AuditFinding(String dimension, String description, String suggestedFix) {}

record AuditFindings(List<AuditFinding> findings, String overallRationale) {}

record AuditVerdict(CriticDecision decision, AuditFindings findings, int severityScore, Instant auditedAt) {}

record RevisedResponse(String text, Instant revisedAt) {}

record RevisionCycle(
    int cycleNumber,
    ChatbotResponse input,
    GuardrailVerdict guardrail,
    Optional<AuditVerdict> verdict,
    Optional<RevisedResponse> revision
) {}

record Session(
    String sessionId,
    String channel,
    int maxRevisions,
    SessionStatus status,
    List<RevisionCycle> cycles,
    Optional<Integer> approvedAtCycle,
    Optional<String> approvedText,
    Optional<String> escalationReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { AUDITING, REVISING, APPROVED, ESCALATED }

enum CriticDecision { PASS, REVISE }
```

### Events (on `SessionEntity`)

`SessionCreated`, `CycleStarted`, `GuardrailVerdictRecorded`, `AuditVerdictRecorded`, `RevisionProduced`, `SessionApproved`, `SessionEscalated`, `AuditRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ text, channel?, maxRevisions? }` → `{ sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=AUDITING|REVISING|APPROVED|ESCALATED`.
- `GET /api/sessions/{id}` — one session (including every revision cycle and every finding set).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "LLM Auditor"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a response text and channel, live list of sessions with status pills, click-to-expand per-cycle timeline showing each revision, the guardrail verdict, the critic's verdict, and the audit findings.

Browser title: `<title>Akka Sample: LLM Auditor</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — response guardrail** (`after-llm-response`): a deterministic check that the inbound response's initial severity score (assigned by a keyword + rule scan) does not exceed the blocking threshold (default: severity ≥ 9 on a 0–10 scale). Responses above the threshold are immediately escalated with `reasonCode = SEVERITY_EXCEEDS_THRESHOLD` and never enter the revision loop. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's audit verdict is recorded as an `AuditRecorded` event with `{ cycleNumber, decision, severityScore, guardrailBlocked }`. The `AuditSampler` TimedAction is the canonical writer; the workflow also emits an event on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-cycle timeline and in `/api/sessions/{id}`.

## 9. Agent prompts

- `CriticAgent` → `prompts/critic.md`. Audits a chatbot response against the four-dimension rubric; returns `PASS` with a rationale or `REVISE` with up to four typed findings.
- `ReviserAgent` → `prompts/reviser.md`. Rewrites a response, addressing every finding in the audit before returning; does not change factual content that was already correct.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a response; session progresses `AUDITING` → `REVISING` → `APPROVED` within the revision ceiling; the App UI shows every cycle's revision and findings.
2. **J2 — halt at ceiling** — Submit a response whose rubric problems are impossible to resolve (test mode forces the Critic to `REVISE` every cycle); session progresses through every revision and lands in `ESCALATED` with the lowest-severity revision preserved.
3. **J3 — guardrail block** — Submit a response with a severity-triggering keyword; the guardrail escalates immediately with `reasonCode = SEVERITY_EXCEEDS_THRESHOLD`; no revision cycle is started.
4. **J4 — eval-event timeline** — The expanded view of any session shows one `AuditRecorded` event per completed cycle and one terminal event on disposition.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named llm-auditor demonstrating the evaluator-optimizer ×
governance-risk cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-governance-risk-llm-auditor.
Java package io.akka.samples.llmauditor. Akka 3.6.0. HTTP port 9503.

Components to wire (exactly):
- 2 AutonomousAgents:
  * CriticAgent — definition() with
    capability(TaskAcceptance.of(AUDIT).maxIterationsPerTask(2)).
    System prompt loaded from prompts/critic.md. Returns AuditVerdict{decision,
    findings, severityScore, auditedAt} where decision is the CriticDecision
    enum (PASS | REVISE) and severityScore is a 0–10 integer.
  * ReviserAgent — definition() with
    capability(TaskAcceptance.of(REVISE_RESPONSE).maxIterationsPerTask(3)).
    System prompt from prompts/reviser.md. Returns RevisedResponse{text,
    revisedAt}. The REVISE_RESPONSE task takes (originalResponse, findings:
    AuditFindings) as inputs.

- 1 Workflow AuditWorkflow with steps:
    startStep -> guardrailStep -> [severity >= threshold? escalateStep :
    auditStep] -> [decision PASS? approveStep : (cycleCount < maxRevisions ?
       reviseStep -> auditStep : escalateStep)] -> END.
  guardrailStep is a pure-function step (no LLM call): scans the response
    text for keyword indicators, computes an initialSeverity (0–10), and
    compares against llm-auditor.audit.severity-blocking-threshold (default
    9). On BLOCK, emits GuardrailVerdictRecorded with verdict.passed = false
    and reasonCode = "SEVERITY_EXCEEDS_THRESHOLD", then transitions to
    escalateStep.
  auditStep calls forAutonomousAgent(CriticAgent.class, sessionId).runSingleTask(
    AUDIT) then forTask(taskId).result(AUDIT).
  reviseStep calls forAutonomousAgent(ReviserAgent.class, sessionId)
    .runSingleTask(REVISE_RESPONSE).
  approveStep emits SessionApproved.
  escalateStep emits SessionEscalated with the lowest-severity cycle's text
    as best-of and a structured escalationReason. Override settings() with
    stepTimeout(60s) on auditStep and reviseStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(escalateStep)).

- 1 EventSourcedEntity SessionEntity holding state Session{sessionId, channel,
  maxRevisions, SessionStatus status, List<RevisionCycle> cycles,
  Optional<Integer> approvedAtCycle, Optional<String> approvedText,
  Optional<String> escalationReason, Instant createdAt, Optional<Instant>
  finishedAt}. SessionStatus enum: AUDITING, REVISING, APPROVED, ESCALATED.
  Events: SessionCreated, CycleStarted, GuardrailVerdictRecorded,
  AuditVerdictRecorded, RevisionProduced, SessionApproved, SessionEscalated,
  AuditRecorded. Commands: createSession, startCycle, recordGuardrail,
  recordVerdict, recordRevision, approve, escalate, recordAuditEvent,
  getSession. emptyState() returns Session.initial("", "", 3) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity ResponseQueue with command enqueueResponse(text,
  channel) emitting ResponseReceived{sessionId, text, channel, receivedAt}.

- 1 View SessionsView with row type SessionRow (mirrors Session; the cycles
  list is bounded at maxRevisions so size stays reasonable). Table updater
  consumes SessionEntity events. ONE query getAllSessions SELECT * AS sessions
  FROM sessions_view. No WHERE status filter — caller filters client-side
  because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer ResponseIngestConsumer subscribed to ResponseQueue events; on
  ResponseReceived starts an AuditWorkflow with the sessionId as the workflow
  id.

- 2 TimedActions:
  * ResponseSimulator — every 60s, reads next line from
    src/main/resources/sample-events/chatbot-responses.jsonl and calls
    ResponseQueue.enqueueResponse.
  * AuditSampler — every 30s, queries SessionsView.getAllSessions, finds
    sessions with a completed audit cycle that has not yet been recorded as
    an AuditRecorded event, and calls SessionEntity.recordAuditEvent(
    cycleNumber, decision, severityScore, guardrailBlocked).
    Idempotent per (sessionId, cycleNumber).

- 2 HttpEndpoints:
  * AuditEndpoint at /api with POST /sessions, GET /sessions, GET /sessions/{id},
    GET /sessions/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /sessions body
    is {text, channel?, maxRevisions?}; missing channel defaults to "default",
    missing maxRevisions defaults to 3.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- AuditTasks.java declaring two Task<R> constants: AUDIT (resultConformsTo
  AuditVerdict), REVISE_RESPONSE (RevisedResponse).
- Domain records ChatbotResponse, GuardrailVerdict, AuditFinding, AuditFindings,
  AuditVerdict, RevisedResponse, RevisionCycle, Session; enums SessionStatus,
  CriticDecision.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9503 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  llm-auditor.audit.max-revisions = 3 and
  llm-auditor.audit.severity-blocking-threshold = 9, overridable by env var.
- src/main/resources/sample-events/chatbot-responses.jsonl with 8 canned
  response lines, each shaped {"text":"...", "channel":"support"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 response guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = chatbot-response-auditing,
  decisions.authority_level = advisory, data.data_classes.pii = false,
  capabilities.content-generation = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/critic.md, prompts/reviser.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: LLM Auditor",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-cycle timeline). Browser title exactly:
  <title>Akka Sample: LLM Auditor</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: critic.json, reviser.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    critic.json — 6 AuditVerdict entries. Three return decision=PASS with
      severityScore=1 or 2 and a one-sentence rationale. Three return
      decision=REVISE with severityScore=5 or 6 and an AuditFindings payload
      with two findings (one tone finding, one hallucination-risk finding).
    reviser.json — 6 RevisedResponse entries. Three are revised responses
      that address the tone finding ("removed speculative phrasing; added
      factual citation placeholder"). Two address hallucination risk
      ("replaced unverified statistic with hedged claim"). One is an
      intentionally un-improved response used to exercise the escalation
      path in J2.
- A MockModelProvider.seedFor(sessionId, cycleNumber) helper makes the
  selection deterministic per (sessionId, cycleNumber) so the same session
  in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. CriticAgent
  and ReviserAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an AuditTasks companion declaring the two Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Session row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: AuditTasks.java is mandatory; generating CriticAgent or
  ReviserAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9503, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
