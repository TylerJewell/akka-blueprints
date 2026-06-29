# SPEC — retention-outreach

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Student Retention Outreach Agent.
**One-line pitch:** A background worker monitors student engagement signals, redacts FERPA-sensitive data, classifies learners by attrition risk, and holds every outreach draft for advisor approval before sending.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of a single AI primitive (`OutreachDraftAgent`). Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw engagement event and any LLM call — the model never sees student names, email addresses, student ID numbers, or addresses. This satisfies FERPA's minimum-necessary principle at the architectural level, not as a prompt instruction.
- A **HITL approve-before-send** gate is the primary production safeguard: the AI drafts outreach but cannot send it. Only an authenticated advisor can transition a drafted alert to SENT.

The result is a system where every outreach message has had two independent checks (PII redaction, human approval) before it reaches a student.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live alert board: every engagement signal received, its sanitized payload, its risk classification, and (if drafted) the proposed outreach message.
2. `EngagementPoller` (TimedAction) ticks every 15 s and inserts new simulated engagement events into `EngagementEventQueue`. (A simulator style — drips canned records.)
3. For each new event: `PiiSanitizer` (Consumer) redacts the payload, then `AtRiskClassifierAgent` classifies it.
4. If the classification is `AT_RISK`, `OutreachDraftAgent` produces an outreach message. The alert lands in `AWAITING_APPROVAL`.
5. The advisor clicks Approve in the UI — the alert transitions to SENT (simulated; no real email is dispatched).
6. The advisor clicks Reject in the UI with a reason — the alert transitions to DISMISSED.
7. `EvalRunner` (TimedAction) ticks every 30 minutes, picks N sent messages without `evalScore`, calls a judge agent, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EngagementPoller` | `TimedAction` | Drips simulated engagement events every 15 s. | scheduler | `EngagementEventQueue` |
| `EngagementEventQueue` | `EventSourcedEntity` | Append-only log of `EngagementSignalReceived` events. | `EngagementPoller`, `OutreachEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Reads `EngagementSignalReceived` events, redacts FERPA-sensitive fields, emits `SignalSanitized` via `StudentOutreachEntity`. | `EngagementEventQueue` events | `StudentOutreachEntity` |
| `AtRiskClassifierAgent` | `Agent` (typed, not autonomous) | Classifies sanitized engagement payload into `AT_RISK` / `MONITOR` / `ON_TRACK`. | invoked by Workflow | returns `ClassificationResult` |
| `OutreachDraftAgent` | `AutonomousAgent` | Drafts an outreach message for `AT_RISK` students. | invoked by Workflow | returns `DraftedOutreach` |
| `StudentOutreachWorkflow` | `Workflow` | Per-alert orchestration: classify → (maybe draft) → wait for advisor. | `PiiSanitizer` (one workflow per `SignalSanitized`) | `StudentOutreachEntity` |
| `StudentOutreachEntity` | `EventSourcedEntity` | Lifecycle per alert: received → sanitized → classified → drafted → approved/rejected → sent/dismissed. | `StudentOutreachWorkflow` | `OutreachView` |
| `OutreachView` | `View` | Read-model row per alert for the UI. | `StudentOutreachEntity` events | `OutreachEndpoint` |
| `EvalRunner` | `TimedAction` | Every 30 min, samples sent alerts without scores; calls judge; writes `EvalScored`. | scheduler | `StudentOutreachEntity` |
| `OutreachEndpoint` | `HttpEndpoint` | `/api/outreach/*` — list, get, approve, reject, SSE. | — | `OutreachView`, `StudentOutreachEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record EngagementSignal(
    String signalId, String studentId, String courseId,
    String signalType, double engagementScore,
    String rawNotes, Instant observedAt
) {}

record SanitizedSignal(
    String redactedNotes, List<String> piiCategoriesFound,
    String signalType, double engagementScore
) {}

record ClassificationResult(RiskLevel level, String confidence, String reason) {}
enum RiskLevel { AT_RISK, MONITOR, ON_TRACK }

record DraftedOutreach(String subject, String body, Instant draftedAt) {}

record AdvisorDecision(
    boolean approved, String decidedBy,
    Optional<String> reason, Instant decidedAt
) {}

record StudentAlert(
    String alertId,
    EngagementSignal signal,
    Optional<SanitizedSignal> sanitized,
    Optional<ClassificationResult> classification,
    Optional<DraftedOutreach> draft,
    Optional<AdvisorDecision> decision,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    AlertStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AlertStatus {
    RECEIVED, SANITIZED, CLASSIFIED, DRAFTED,
    AWAITING_APPROVAL, SENT, DISMISSED, MONITORING
}
```

Events on `StudentOutreachEntity`: `EngagementSignalReceived`, `SignalSanitized`, `AlertClassified`, `OutreachDrafted`, `OutreachApproved`, `OutreachRejected`, `AlertSent`, `AlertDismissed`, `AlertMonitored`, `EvalScored`.

Events on `EngagementEventQueue`: `EngagementSignalReceived` (re-emitted as the audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/outreach` — list all alerts. Optional `?status=…`.
- `GET /api/outreach/{id}` — one alert.
- `POST /api/outreach/{id}/approve` — body `{ advisorId }` → transitions AWAITING_APPROVAL to SENT.
- `POST /api/outreach/{id}/reject` — body `{ advisorId, reason }` → transitions to DISMISSED.
- `GET /api/outreach/sse` — Server-Sent Events for every alert change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Student Retention Outreach Agent</title>`.

App UI tab is the most distinctive: it shows the **live alert board** with a filterable list on the left and a selected-alert detail pane on the right. The detail pane shows the sanitized engagement summary, the classification reason, the drafted outreach message (editable), and Approve/Reject buttons.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts student names, email addresses, student ID numbers, phone numbers, and addresses from engagement notes before any LLM call. Records the categories found for audit.
- **H1 — HITL approve-before-send** gate: the workflow explicitly pauses in AWAITING_APPROVAL; only `OutreachEndpoint`'s approve/reject endpoints can advance it. Advisors review every outreach draft.

## 9. Agent prompts

- `AtRiskClassifierAgent` → `prompts/at-risk-classifier.md`. Typed classifier. Always returns one of the three `RiskLevel` values.
- `OutreachDraftAgent` → `prompts/outreach-draft.md`. Drafts an advisor outreach message using only the sanitized signal — never the raw engagement record.
- `EvalJudge` (used by `EvalRunner`) → `prompts/eval-judge.md`. Scores a sent outreach message on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an engagement signal; it appears in the UI within 15 s; passes sanitize → classify → draft → AWAITING_APPROVAL.
2. **J2** — Advisor clicks Approve; alert transitions to SENT; simulated send is logged but no network call leaves the process.
3. **J3** — Advisor clicks Reject with reason; alert transitions to DISMISSED with the reason visible.
4. **J4** — Eval Runner scores at least one SENT alert within 30 minutes; score and rationale appear in the UI.
5. **J5** — PII audit: raw student identifiers from the engagement signal never appear in the LLM call payload.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named retention-outreach demonstrating the continuous-monitor × education
cell. Runs out of the box (in-memory engagement simulator; no real LMS integration). Maven group
io.akka.samples. Artifact continuous-monitor-education-retention-outreach. Java package
io.akka.samples.studentretentionoutreachagent. HTTP port 9172.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) AtRiskClassifierAgent — classifier. System prompt loaded from
  prompts/at-risk-classifier.md. Input: SanitizedSignal{redactedNotes, piiCategoriesFound:
  List<String>, signalType: String, engagementScore: double}. Output:
  ClassificationResult{level: RiskLevel (enum AT_RISK/MONITOR/ON_TRACK), confidence:
  "high"|"medium"|"low", reason: String}. Defaults to AT_RISK under uncertainty.

- 1 AutonomousAgent OutreachDraftAgent — definition() with capability(TaskAcceptance.of(DRAFT)
  .maxIterationsPerTask(3)). System prompt from prompts/outreach-draft.md. Input:
  SanitizedSignal + Optional<String> advisorContext. Output: DraftedOutreach{subject, body,
  draftedAt}. The agent NEVER calls a send tool directly.

- 1 AutonomousAgent EvalJudge — definition() with capability(TaskAcceptance.of(EVAL)
  .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md. Input: SanitizedSignal
  + DraftedOutreach. Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow StudentOutreachWorkflow per alert with steps: classifyStep -> conditional branch:
  AT_RISK -> draftStep -> awaitApprovalStep -> finaliseStep; MONITOR -> ends with
  AlertMonitored; ON_TRACK -> ends with AlertDismissed(reason="filter:ON_TRACK").
  classifyStep wraps AtRiskClassifierAgent call with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(10)). draftStep wraps OutreachDraftAgent with stepTimeout 30s.
  awaitApprovalStep polls StudentOutreachEntity.getAlert every 5s; on decision.isPresent()
  advances. No auto-timeout on awaitApproval — drafts wait indefinitely. finaliseStep emits
  AlertSent or AlertDismissed based on decision.approved.

- 2 EventSourcedEntities:
  * EngagementEventQueue — append-only audit log. Command receive(EngagementSignal) emits
    EngagementSignalReceived{signal}.
  * StudentOutreachEntity (one per alertId) — full per-alert lifecycle. State
    StudentAlert{alertId, signal: EngagementSignal{signalId, studentId, courseId, signalType,
    engagementScore, rawNotes, observedAt}, Optional<SanitizedSignal> sanitized,
    Optional<ClassificationResult> classification, Optional<DraftedOutreach> draft,
    Optional<AdvisorDecision> decision (with approved: boolean, decidedBy,
    Optional<String> reason, decidedAt), Optional<Integer> evalScore,
    Optional<String> evalRationale, AlertStatus status, Instant createdAt,
    Optional<Instant> finishedAt}. AlertStatus enum: RECEIVED, SANITIZED, CLASSIFIED,
    DRAFTED, AWAITING_APPROVAL, SENT, DISMISSED, MONITORING. Events:
    EngagementSignalReceived, SignalSanitized, AlertClassified, OutreachDrafted,
    OutreachApproved, OutreachRejected, AlertSent, AlertDismissed, AlertMonitored,
    EvalScored. Commands: registerSignal, attachSanitized, attachClassification,
    attachDraft, recordApproval, recordRejection, markSent, markDismissed,
    markMonitoring, recordEval, getAlert. emptyState() returns StudentAlert.initial("", null)
    without commandContext() reference.

- 1 Consumer PiiSanitizer subscribed to EngagementEventQueue events; for each
  EngagementSignalReceived, applies a regex+heuristic redaction pipeline (student emails,
  ID numbers, phone numbers, names in notes, addresses) to rawNotes, builds SanitizedSignal
  with piiCategoriesFound, and calls StudentOutreachEntity.registerSignal followed by
  attachSanitized. Then starts a StudentOutreachWorkflow with alertId as the workflow id.

- 1 View OutreachView with row type StudentAlertRow (mirrors StudentAlert minus raw signal
  rawNotes). Table updater consumes StudentOutreachEntity events. ONE query getAllAlerts
  SELECT * AS alerts FROM outreach_view. No WHERE status filter — caller filters client-side.

- 2 TimedActions:
  * EngagementPoller — every 15s, reads next line from src/main/resources/sample-events/
    engagement-signals.jsonl and calls EngagementEventQueue.receive.
  * EvalRunner — every 30 minutes, queries OutreachView.getAllAlerts, picks up to 5 SENT
    alerts without an evalScore (oldest-first), calls EvalJudge with a 1–5 rubric per
    alert, then calls StudentOutreachEntity.recordEval(score, rationale) per alert.

- 2 HttpEndpoints:
  * OutreachEndpoint at /api with GET /outreach, GET /outreach/{id}, POST /outreach/{id}/approve
    (body {advisorId}), POST /outreach/{id}/reject (body {advisorId, reason}),
    GET /outreach/sse, and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Approve writes OutreachApproved + AlertSent; reject writes
    OutreachRejected + AlertDismissed.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- OutreachTasks.java declaring three Task<R> constants: CLASSIFY (ClassificationResult),
  DRAFT (DraftedOutreach), EVAL (EvalResult).
- Domain records EngagementSignal, SanitizedSignal, ClassificationResult, DraftedOutreach,
  AdvisorDecision, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9172 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/engagement-signals.jsonl with 8 canned records covering
  AT_RISK (multiple missed sessions, sharply declining scores), MONITOR (mild drop),
  and ON_TRACK (normal participation) classifications.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, H1 hitl application.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = draft-only,
  oversight.human_in_loop = true, oversight.reviewer_must_approve_every_outgoing = true,
  failure.failure_modes including "pii-leakage-via-llm" and "missed-at-risk-student";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/at-risk-classifier.md, prompts/outreach-draft.md, prompts/eval-judge.md loaded as
  agent system prompts.
- README.md at the project root: title "Akka Sample: Student Retention Outreach Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live alert list with risk-level chips and PII category chips; right = selected alert
  detail with editable outreach draft textarea + Approve/Reject buttons). Browser title exactly:
  <title>Akka Sample: Student Retention Outreach Agent</title>. No subtitle on Overview tab.

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
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    at-risk-classifier.json — 8–12 ClassificationResult entries spanning
      AT_RISK (multiple missed sessions, sharply declining engagement scores,
      unresponsive to prior contact), MONITOR (single missed session, mild
      score drop), and ON_TRACK (normal participation, engagement score above
      threshold). Each entry has confidence and a one-sentence reason. The
      mock should AT_RISK under ambiguity — match the prompt's default
      behaviour.
    outreach-draft.json — 4–6 DraftedOutreach entries with subject prefixed
      "Checking in:", 3–4 paragraph bodies, signed "— Academic Advising".
      No invented policy timelines; no echoed [REDACTED] tokens.
    eval-judge.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric (tone / accuracy / policy adherence).
- A MockModelProvider.seedFor(alertId) helper makes per-alert selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 are load-bearing for this pattern.
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside an Agent's prompt
  and not after the LLM has seen the raw payload. FERPA compliance depends on this ordering.
- The agent NEVER sends; only the human Approve endpoint can transition to SENT.
- The generated static-resources/index.html must include the mermaid CSS overrides AND theme
  variables from Lesson 24 (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state names render
  black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel attribute,
  NEVER by NodeList index (Lesson 26). No hidden zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
