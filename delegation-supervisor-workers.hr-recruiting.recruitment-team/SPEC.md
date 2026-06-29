# Recruitment Team — `recruitment-team`

This file is the natural-language input for `/akka:specify`. Run `/akka:specify @SPEC.md` from inside the folder. Sections 1–11 are the spec; Section 12 chains the rest of the workflow.

---

## 1. System name + one-line pitch

**Recruitment Team.** A candidate application arrives; the system redacts protected attributes, a supervisor agent delegates matching and screening to worker agents, every screen decision is evaluated, and a human approves or rejects before any candidate outcome is final.

## 2. What this blueprint demonstrates

The coordination pattern is **delegation-supervisor-workers**: a `ScreeningWorkflow` delegates to two worker agents (`MatchAgent`, `ScreenAgent`) and a `SupervisorAgent` that aggregates their typed outputs into one recommendation. The governance pattern wires four controls: a sanitizer redacts protected attributes from resume text before any agent sees it, an event-eval scores each screen decision, a periodic fairness/drift watch samples recent decisions, and a human-in-loop gate holds the final candidate outcome for review.

## 3. User-facing flows

1. A recruiter (or the simulator) submits an application with a role id and resume text. The system returns a `candidateId`.
2. The workflow redacts protected attributes (age, gender, ethnicity signals) and records `ResumeSanitized`. Status becomes `SANITIZED`.
3. `MatchAgent` scores fit against the role; `ScreenAgent` produces an advance/hold/reject screen; `SupervisorAgent` aggregates both. Status moves through `MATCHED`, `SCREENED`, then `AWAITING_DECISION`.
4. The App UI lists the candidate with the recommendation and Approve/Reject buttons.
5. A human clicks Approve or Reject (with a reason). Status becomes `APPROVED` or `REJECTED`.
6. A decision left untouched past the wait window escalates to `ESCALATED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MatchAgent` | AutonomousAgent | Score candidate-to-role fit | `ScreeningWorkflow` | `CandidateEntity` |
| `ScreenAgent` | AutonomousAgent | Advance/hold/reject screen | `ScreeningWorkflow` | `CandidateEntity` |
| `SupervisorAgent` | AutonomousAgent | Aggregate worker output into one recommendation | `ScreeningWorkflow` | `CandidateEntity` |
| `ScreeningWorkflow` | Workflow | Delegate sanitize → match → screen → supervise → await → finalize | `ApplicationConsumer` | agents, `CandidateEntity` |
| `CandidateEntity` | EventSourcedEntity | Candidate lifecycle state | `ScreeningWorkflow`, endpoint | `CandidatesView` |
| `InboundApplicationQueue` | EventSourcedEntity | Inbound application buffer | `ScreeningEndpoint`, `ApplicationSimulator` | `ApplicationConsumer` |
| `CandidatesView` | View | CQRS read model of candidates | `CandidateEntity` | `ScreeningEndpoint` |
| `ApplicationConsumer` | Consumer | Start a workflow per application | `InboundApplicationQueue` | `ScreeningWorkflow` |
| `ApplicationSimulator` | TimedAction | Drip canned applications | — | `InboundApplicationQueue` |
| `FairnessDriftMonitor` | TimedAction | Sample recent decisions for skew/drift | `CandidatesView` | `CandidateEntity` |
| `StuckDecisionMonitor` | TimedAction | Escalate stale awaiting decisions | `CandidatesView` | `CandidateEntity` |
| `ScreeningEndpoint` | HttpEndpoint | REST + SSE + metadata | UI | entities, view |
| `AppEndpoint` | HttpEndpoint | Serve static UI | browser | static-resources |

## 5. Data model

Authoritative; see [`reference/data-model.md`](reference/data-model.md). Records (nullable lifecycle fields are `Optional<T>` per Lesson 6):

- `Candidate(String id, String roleId, Optional<String> sanitizedResume, CandidateStatus status, Optional<Integer> matchScore, Optional<String> matchSummary, Optional<String> screenRecommendation, Optional<String> screenReasons, Optional<Integer> evalScore, Optional<String> supervisorSummary, Optional<String> decidedBy, Optional<String> decisionReason, Optional<String> sanitizedAt, Optional<String> screenedAt, Optional<String> decidedAt, Optional<String> escalatedAt, Optional<Boolean> driftFlagged)`
- `CandidateStatus` enum: `RECEIVED, SANITIZED, MATCHED, SCREENED, AWAITING_DECISION, APPROVED, REJECTED, ESCALATED`.
- Typed agent results: `CandidateMatch(int score, String summary, List<String> matchedSkills)`, `ScreenDecision(String recommendation, String reasons)`, `SupervisorSummary(String recommendation, String summary)`.
- Events: `ResumeSanitized`, `CandidateMatched`, `CandidateScreened`, `DecisionEvaluated`, `DecisionRequested`, `CandidateApproved`, `CandidateRejected`, `CandidateEscalated`, `DriftFlagged`.

## 6. API contract

Inline surface; payload schemas in [`reference/api-contract.md`](reference/api-contract.md).

```
POST /api/applications                  -> { candidateId }
POST /api/candidates/{id}/approve       -> 200 | 404
POST /api/candidates/{id}/reject        -> 200 | 404
GET  /api/candidates                    -> { candidates: [Candidate, ...] }
GET  /api/candidates/{id}               -> Candidate
GET  /api/candidates/sse                -> Server-Sent Events of Candidate
GET  /api/metadata/eval-matrix          -> text/yaml
GET  /api/metadata/risk-survey          -> text/yaml
GET  /api/metadata/readme               -> text/markdown
GET  /                                   -> 302 /app/index.html
GET  /app/{*path}                        -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/`, no npm. Browser title `<title>Akka Sample: Recruitment Team</title>`. Five tabs (Overview / Architecture / Risk Survey / Eval Matrix / App UI). Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index, with no zombie panels (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label and edge-label CSS overrides (Lesson 24). Details in [`reference/ui-mockup.md`](reference/ui-mockup.md). The App UI tab submits an application, streams the candidate list via SSE, and shows Approve/Reject on candidates in `AWAITING_DECISION`.

## 8. Governance

Controls in [`eval-matrix.yaml`](eval-matrix.yaml); deployer survey in [`risk-survey.yaml`](risk-survey.yaml).

- **S1 sanitizer / special-category** — `ResumeSanitizer` redacts protected-attribute signals from resume text in `sanitizeStep` before any agent call; the workflow cannot reach `matchStep` until sanitization records `ResumeSanitized`.
- **E1 eval-event / on-decision-eval** — after `ScreenAgent` returns, `screenStep` scores the decision and records `DecisionEvaluated` with an eval score surfaced in the UI.
- **P1 eval-periodic / drift-fairness-watch** — `FairnessDriftMonitor` samples recent decisions on a timer, computes a skew signal, and records `DriftFlagged` when it crosses a threshold.
- **H1 hitl / application** — `awaitDecisionStep` pauses; the candidate sits in `AWAITING_DECISION` until a human approve/reject command arrives.

## 9. Agent prompts

- `MatchAgent` — score candidate-to-role fit from sanitized resume and role. See [`prompts/match-agent.md`](prompts/match-agent.md).
- `ScreenAgent` — produce an advance/hold/reject screen with reasons. See [`prompts/screen-agent.md`](prompts/screen-agent.md).
- `SupervisorAgent` — aggregate match and screen into one recommendation. See [`prompts/supervisor-agent.md`](prompts/supervisor-agent.md).

## 10. Acceptance

Full journeys in [`reference/user-journeys.md`](reference/user-journeys.md).

1. **Submit and watch a recommendation.** POST an application; within ~30 s the candidate appears in `AWAITING_DECISION` with a non-empty supervisor recommendation and an eval score.
2. **Approve a candidate.** Approve in the App UI; status becomes `APPROVED` with `decidedBy` set.
3. **Reject a candidate.** Reject with a reason; status becomes `REJECTED` and the reason renders.
4. **Drift watch flags skew.** With the simulator seeding applications, `FairnessDriftMonitor` sets `driftFlagged` on at least one candidate when recent decisions skew.

## 11. Implementation directives

```
Create a sample named recruitment-team demonstrating the
delegation-supervisor-workers x hr-recruiting cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
recruitment-team. Java package io.akka.samples.recruitmentteam. Akka 3.6.0.
HTTP port 9654.

Components to wire (exactly):
- 3 AutonomousAgents: MatchAgent (returns CandidateMatch{score, summary,
  matchedSkills}), ScreenAgent (returns ScreenDecision{recommendation,
  reasons}), SupervisorAgent (returns SupervisorSummary{recommendation,
  summary}). Each declares definition() with
  capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Each requires a
  companion Tasks file declaring its Task<R> constant (Lesson 7).
- 1 Workflow ScreeningWorkflow with steps sanitizeStep -> matchStep ->
  screenStep -> superviseStep -> awaitDecisionStep -> finalizeStep. sanitizeStep
  runs ResumeSanitizer (deterministic redaction, no agent) and records
  ResumeSanitized. matchStep/screenStep/superviseStep call
  forAutonomousAgent(...).runSingleTask(...) then forTask(taskId).result(...).
  screenStep also computes an eval score and records DecisionEvaluated.
  awaitDecisionStep polls CandidateEntity.getCandidate; on AWAITING_DECISION
  self-schedules a 5-second resume timer; on APPROVED/REJECTED/ESCALATED ends.
  Override settings() with stepTimeout(60s) on matchStep, screenStep,
  superviseStep (Lesson 4); defaultStepRecovery maxRetries(2) failoverTo an
  error step. WorkflowSettings is nested in Workflow — no import (Lesson 5).
- 1 EventSourcedEntity CandidateEntity holding the Candidate record with the
  status enum and Optional lifecycle fields. Events: ResumeSanitized,
  CandidateMatched, CandidateScreened, DecisionEvaluated, DecisionRequested,
  CandidateApproved, CandidateRejected, CandidateEscalated, DriftFlagged.
  Commands: recordSanitized, recordMatch, recordScreen, recordEval,
  requestDecision, approve, reject, markEscalated, flagDrift, getCandidate.
  emptyState() returns Candidate.initial("","") with NO commandContext()
  reference (Lesson 3).
- 1 EventSourcedEntity InboundApplicationQueue with one command
  enqueueApplication(roleId, resume) emitting ApplicationQueued.
- 1 View CandidatesView with row type Candidate, table updater consuming
  CandidateEntity events. ONE query getAllCandidates SELECT * AS candidates
  FROM candidates_view. No WHERE status filter (Akka cannot auto-index enum
  columns, Lesson 2) — filter client-side in callers.
- 1 Consumer ApplicationConsumer subscribed to InboundApplicationQueue events;
  on each event starts a ScreeningWorkflow with a fresh UUID.
- 3 TimedActions: ApplicationSimulator (every 30s, reads next line from
  src/main/resources/sample-events/applications.jsonl and calls
  InboundApplicationQueue.enqueueApplication); FairnessDriftMonitor (every 45s,
  queries CandidatesView.getAllCandidates, computes a skew signal over recent
  decided candidates, calls CandidateEntity.flagDrift when it crosses
  threshold); StuckDecisionMonitor (every 30s, finds AWAITING_DECISION older
  than 2 min and calls CandidateEntity.markEscalated).
- 2 HttpEndpoints: ScreeningEndpoint at /api (applications, approve, reject,
  candidates list filtered client-side from getAllCandidates, single candidate,
  SSE stream, /api/metadata/* serving the YAML/MD files from
  src/main/resources/metadata/). AppEndpoint serving / -> 302 /app/index.html
  and /app/* -> static-resources/*.
- 1 ServiceSetup.

Companion files:
- ScreeningTasks.java (or per-agent Tasks files) declaring Task<R> constants:
  MATCH (resultConformsTo CandidateMatch), SCREEN (ScreenDecision), SUPERVISE
  (SupervisorSummary).
- ResumeSanitizer.java — deterministic redaction of protected-attribute signals
  (names, ages, gendered pronouns, ethnicity/nationality tokens) to placeholder
  tokens.
- Records: CandidateMatch(int score, String summary, List<String>
  matchedSkills), ScreenDecision(String recommendation, String reasons),
  SupervisorSummary(String recommendation, String summary).
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9654 and model-provider blocks for
  anthropic (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
  (gemini-2.5-flash), keys read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify model names are current before locking
  (Lesson 8).
- src/main/resources/sample-events/applications.jsonl with 8 canned
  application lines (varied roles and resume profiles).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root files the endpoint serves from classpath).
- eval-matrix.yaml and risk-survey.yaml at the project root.
- README.md at the project root (no governance-mechanisms section, no
  configuration section).
- src/main/resources/static-resources/index.html — single self-contained file
  (Lesson 17). Five tabs. Mermaid CSS overrides (Lesson 24). Attribute-based
  tab switching with no zombie panels (Lesson 26). Match the governance.html
  visual style (dark / yellow accent / Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (MatchAgent -> CandidateMatch, ScreenAgent -> ScreenDecision,
  SupervisorAgent -> SupervisorSummary; write 4-6 entries each to
  src/main/resources/mock-responses/{match-agent,screen-agent,supervisor-agent}.json).
  Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session
  end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, or secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes any captured key.

Constraints — see AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: explicit stepTimeout on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on the Candidate row record.
- Lesson 7: each AutonomousAgent has a companion Tasks declaration.
- Lesson 8: verify model names are current.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9654.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column, no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box", never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-label colour + edge-label foreignObject
  overflow:visible + transitionLabelColor #cccccc.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: attribute-based tab switching; delete dead panels, never
  display:none them.
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the SDD spec into `specs/features/`, **do not stop and wait for the user**. Continue automatically:

1. Run `/akka:plan` — produce the plan; accept defaults, pick the most conservative option for anything ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into tasks; same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. On a task failure, capture it, continue with the next task, and summarise all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9654/`) and a one-line summary of any failures from step 3.

**Stop earlier only** for a hard error you cannot work around: an unresolved API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fixes, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
