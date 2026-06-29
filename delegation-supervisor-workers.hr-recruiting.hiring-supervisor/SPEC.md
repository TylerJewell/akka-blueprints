# SPEC — hiring-supervisor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Hiring Supervisor Orchestration.
**One-line pitch:** Submit a candidate application; a supervisor dispatches screening and scheduling to specialist agents in parallel, then consolidates a structured hiring recommendation.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to consolidate a final recommendation. The blueprint also demonstrates a **before-agent-invocation guardrail** that checks policy before every supervisor delegate call, and a **special-category sanitizer** that strips protected-attribute fields from the application payload before any agent processes it.

## 3. User-facing flows

The user opens the App UI tab and submits a candidate application via the form.

1. The system creates a `CandidateApplication` record in `RECEIVED` and starts a `HiringWorkflow`.
2. The sanitizer strips any special-category fields (gender, age, disability indicators) from the raw payload before it enters the agent pipeline.
3. The supervisor receives the sanitized application, checks delegation policy, then decomposes work into a screening task for ScreeningAgent and a scheduling task for SchedulingAgent.
4. The workflow forks: both agents run concurrently. Each returns a typed payload.
5. HiringSupervisor consolidates the two payloads into a `HiringRecommendation { decision, screeningScore, proposedSlots, rationale, guardrailVerdict }`.
6. If a policy check fires before any delegate call, the application moves to `POLICY_HOLD`. Otherwise the application moves to `RECOMMENDED` or `REJECTED` based on the supervisor's decision.
7. If either worker times out after 60 seconds, the workflow short-circuits: the supervisor consolidates from whichever side returned, and the application enters `DEGRADED`.

A `CandidatePipelineSimulator` (TimedAction) drips a sample application every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `HiringSupervisor` | `AutonomousAgent` | Plans evaluation stages, runs the before-invocation guardrail, consolidates sub-agent results into a hiring recommendation. | `HiringWorkflow` | returns typed result to workflow |
| `ScreeningAgent` | `AutonomousAgent` | Evaluates candidate qualifications against job requirements; returns a scored screening report. | `HiringWorkflow` | — |
| `SchedulingAgent` | `AutonomousAgent` | Proposes interview slots given the candidate's role tier and availability window. | `HiringWorkflow` | — |
| `HiringWorkflow` | `Workflow` | Coordinates sanitization, parallel fan-out, consolidation, and persistence. | `HiringEndpoint`, `ApplicationConsumer` | `ApplicationEntity` |
| `ApplicationEntity` | `EventSourcedEntity` | Holds the application's lifecycle (received → under review → recommended / rejected / degraded / policy-hold). | `HiringWorkflow` | `ApplicationView` |
| `ApplicationQueue` | `EventSourcedEntity` | Logs each submitted application for replay and audit. | `HiringEndpoint`, `CandidatePipelineSimulator` | `ApplicationConsumer` |
| `ApplicationView` | `View` | List-of-applications read model. | `ApplicationEntity` events | `HiringEndpoint` |
| `ApplicationConsumer` | `Consumer` | Listens to `ApplicationQueue` events and starts a workflow per submission. | `ApplicationQueue` events | `HiringWorkflow` |
| `CandidatePipelineSimulator` | `TimedAction` | Drips a sample application every 90 s. | scheduler | `ApplicationQueue` |
| `EvalSampler` | `TimedAction` | Samples one recommended application every 5 minutes for eval scoring; emits an `EvalScored` event. | scheduler | `ApplicationEntity` |
| `HiringEndpoint` | `HttpEndpoint` | `/api/hiring/*` — submit, get, list, SSE. | — | `ApplicationView`, `ApplicationQueue`, `ApplicationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ApplicationRequest(String candidateId, String roleId, String resumeText,
                          String availabilityWindow, String requestedBy) {}

record SanitizedApplication(String candidateId, String roleId, String resumeText,
                            String availabilityWindow, boolean sanitized) {}

record ScreeningReport(String candidateId, int qualificationScore,
                       List<String> strengthHighlights, List<String> gapNotes,
                       Instant screenedAt) {}

record ProposedSlot(String slotId, Instant startTime, Instant endTime, String interviewFormat) {}

record SchedulingProposal(String candidateId, List<ProposedSlot> slots, Instant proposedAt) {}

record DelegationPlan(String screeningQuery, String schedulingContext) {}

record HiringRecommendation(String decision, int screeningScore,
                            List<ProposedSlot> proposedSlots, String rationale,
                            String guardrailVerdict, Instant consolidatedAt) {}

record CandidateApplication(
    String applicationId,
    String candidateId,
    String roleId,
    ApplicationStatus status,
    Optional<ScreeningReport> screeningReport,
    Optional<SchedulingProposal> schedulingProposal,
    Optional<HiringRecommendation> recommendation,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ApplicationStatus {
    RECEIVED, UNDER_REVIEW, RECOMMENDED, REJECTED, DEGRADED, POLICY_HOLD
}
```

### Events (on `ApplicationEntity`)

`ApplicationReceived`, `ReviewStarted`, `ScreeningReportAttached`, `SchedulingProposalAttached`, `ApplicationRecommended`, `ApplicationRejected`, `ApplicationDegraded`, `ApplicationPolicyHeld`, `EvalScored`.

### Events (on `ApplicationQueue`)

`ApplicationSubmitted { applicationId, candidateId, roleId, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/hiring` — body `{ candidateId, roleId, resumeText, availabilityWindow }` → `{ applicationId }`. Starts a workflow.
- `GET /api/hiring` — list all applications. Optional `?status=RECEIVED|UNDER_REVIEW|RECOMMENDED|REJECTED|DEGRADED|POLICY_HOLD`.
- `GET /api/hiring/{id}` — one application.
- `GET /api/hiring/sse` — server-sent events stream of every application change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Hiring Supervisor Orchestration"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an application, live list of applications with status pills, expand-row to see screening report + scheduling proposal + recommendation + eval score.

Browser title: `<title>Akka Sample: Hiring Supervisor Orchestration</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-invocation guardrail** (`before-agent-invocation` on `HiringSupervisor`): checks delegation policy before each sub-agent call. Blocking. Failure → `POLICY_HOLD`.
- **S1 — special-category sanitizer** (`special-category` on the workflow's sanitization step): strips protected-attribute fields (gender, age, disability indicators) from the raw `ApplicationRequest` before any agent receives the payload. Blocking at the data-preparation step.

## 9. Agent prompts

- `HiringSupervisor` → `prompts/hiring-supervisor.md`. Plans evaluation stages; checks delegation policy; consolidates screening and scheduling results into a hiring recommendation.
- `ScreeningAgent` → `prompts/screening-agent.md`. Evaluates qualifications; returns `ScreeningReport`.
- `SchedulingAgent` → `prompts/scheduling-agent.md`. Proposes interview slots; returns `SchedulingProposal`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an application; application progresses RECEIVED → UNDER_REVIEW → RECOMMENDED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `ScreeningAgent` timeout to 1 s); application enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a policy-check failure (supervisor returns a flagged delegate call); application enters POLICY_HOLD.
4. **J4** — Wait after a successful recommendation; the application row in the UI shows an eval score.
5. **J5** — Confirm the sanitized payload logged in the entity does not contain protected-attribute fields present in the original request.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hiring-supervisor demonstrating the
delegation-supervisor-workers × hr-recruiting cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-hr-recruiting-hiring-supervisor.
Java package io.akka.samples.hiringsupervisororchestration. Akka 3.6.0. HTTP port 9652.

Components to wire (exactly):
- 3 AutonomousAgents:
  * HiringSupervisor — definition() with capability(TaskAcceptance.of(PLAN_EVALUATION).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/hiring-supervisor.md. Returns DelegationPlan{screeningQuery, schedulingContext}
    for PLAN_EVALUATION and HiringRecommendation{decision, screeningScore, proposedSlots, rationale,
    guardrailVerdict, consolidatedAt} for CONSOLIDATE.
  * ScreeningAgent — capability(TaskAcceptance.of(SCREEN).maxIterationsPerTask(3)). System prompt
    from prompts/screening-agent.md. Returns ScreeningReport{candidateId, qualificationScore,
    strengthHighlights: List<String>, gapNotes: List<String>, screenedAt}.
  * SchedulingAgent — capability(TaskAcceptance.of(SCHEDULE).maxIterationsPerTask(2)). System prompt
    from prompts/scheduling-agent.md. Returns SchedulingProposal{candidateId,
    slots: List<ProposedSlot{slotId, startTime, endTime, interviewFormat}>, proposedAt}.

- 1 Workflow HiringWorkflow with steps:
  sanitizeStep -> planStep -> [parallel] screenStep, scheduleStep -> joinStep -> consolidateStep
  -> guardrailCheckStep -> emitStep.
  sanitizeStep strips special-category fields from the raw ApplicationRequest, producing a
  SanitizedApplication, before any agent invocation.
  planStep calls forAutonomousAgent(HiringSupervisor.class, PLAN_EVALUATION) with the
  SanitizedApplication; the before-agent-invocation guardrail fires here.
  screenStep and scheduleStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(HiringWorkflow::screenStep, ofSeconds(60)) and
  stepTimeout(HiringWorkflow::scheduleStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls consolidateStep with whichever side returned, then ends with
  ApplicationDegraded.
  consolidateStep calls forAutonomousAgent(HiringSupervisor.class, CONSOLIDATE) with the merged
  inputs; give consolidateStep a 90s stepTimeout. guardrailCheckStep runs the deterministic
  policy vetter on the recommendation; on failure, end with ApplicationPolicyHeld. WorkflowSettings
  is nested inside Workflow — no import.

- 1 EventSourcedEntity ApplicationEntity holding state CandidateApplication{applicationId,
  candidateId, roleId, ApplicationStatus, Optional<ScreeningReport> screeningReport,
  Optional<SchedulingProposal> schedulingProposal, Optional<HiringRecommendation> recommendation,
  Optional<String> failureReason, Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant createdAt, Optional<Instant> finishedAt}. ApplicationStatus enum: RECEIVED,
  UNDER_REVIEW, RECOMMENDED, REJECTED, DEGRADED, POLICY_HOLD. Events: ApplicationReceived,
  ReviewStarted, ScreeningReportAttached, SchedulingProposalAttached, ApplicationRecommended,
  ApplicationRejected, ApplicationDegraded, ApplicationPolicyHeld, EvalScored. Commands:
  receiveApplication, startReview, attachScreeningReport, attachSchedulingProposal,
  recommend, reject, degrade, policyHold, recordEval, getApplication.
  emptyState() returns CandidateApplication.initial("", "", null) with no commandContext() reference.

- 1 EventSourcedEntity ApplicationQueue with command submitApplication(candidateId, roleId,
  requestedBy) emitting ApplicationSubmitted{applicationId, candidateId, roleId, requestedBy,
  submittedAt}.

- 1 View ApplicationView with row type ApplicationRow (mirrors CandidateApplication minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  ApplicationEntity events. ONE query getAllApplications SELECT * AS applications FROM application_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer ApplicationConsumer subscribed to ApplicationQueue events; on ApplicationSubmitted
  starts a HiringWorkflow with the applicationId as the workflow id.

- 2 TimedActions:
  * CandidatePipelineSimulator — every 90s, reads next line from
    src/main/resources/sample-events/candidate-applications.jsonl and calls
    ApplicationQueue.submitApplication.
  * EvalSampler — every 5 minutes, queries ApplicationView.getAllApplications, picks the oldest
    RECOMMENDED application without an evalScore, runs a 1-5 rubric judge over the recommendation
    content, then calls ApplicationEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * HiringEndpoint at /api with POST /hiring, GET /hiring, GET /hiring/{id},
    GET /hiring/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- HiringTasks.java declaring four Task<R> constants: PLAN_EVALUATION (DelegationPlan),
  SCREEN (ScreeningReport), SCHEDULE (SchedulingProposal), CONSOLIDATE (HiringRecommendation).
- Domain records ApplicationRequest, SanitizedApplication, DelegationPlan, ProposedSlot,
  ScreeningReport, SchedulingProposal, HiringRecommendation.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9652 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/candidate-applications.jsonl with 8 canned application lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 before-agent-invocation guardrail,
  S1 special-category sanitizer) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector.primary = ["hr-recruiting",
  "talent-acquisition-automation"], decisions.authority_level = recommend-only,
  data.data_classes.special_category = true, capabilities.employment_decisions = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/hiring-supervisor.md, prompts/screening-agent.md, prompts/scheduling-agent.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Hiring Supervisor Orchestration", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Hiring Supervisor Orchestration</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (hiring-supervisor.json,
  screening-agent.json, scheduling-agent.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    hiring-supervisor.json — list of either DelegationPlan or HiringRecommendation objects.
      4–6 DelegationPlan entries (screeningQuery + schedulingContext pairs) and
      4–6 HiringRecommendation entries (each with decision "RECOMMENDED" or "REJECTED",
      a screeningScore 60–95, 1–3 proposedSlots, a 60–100 word rationale, guardrailVerdict = "ok").
    screening-agent.json — 4–6 ScreeningReport entries, each with qualificationScore 50–95,
      2–4 strengthHighlights, and 1–3 gapNotes.
    scheduling-agent.json — 4–6 SchedulingProposal entries, each with 2–3 ProposedSlot objects
      covering the next 5 business days, interviewFormat one of "video", "phone", "on-site".
- A MockModelProvider.seedFor(applicationId) helper makes the selection
  deterministic per application id so the same application produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s consolidation); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion HiringTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9652 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
