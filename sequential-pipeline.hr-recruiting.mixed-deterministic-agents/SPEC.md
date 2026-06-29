# SPEC — score-aggregator

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Non-AI Agents (Score Aggregator).
**One-line pitch:** A candidate application enters the pipeline; `ScreeningAgent` (LLM) screens the resume and generates a recommendation, while `ScoreAggregator` and `StatusNotifier` (deterministic Java) handle scoring and notification — all four phases wired together in a single workflow, with a CI test gate ensuring the deterministic components are verified before any live candidate data enters the system.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an HR recruiting domain, with a deliberate mix of LLM-driven and deterministic Java agents. `ScreeningAgent` (AutonomousAgent) handles the two judgment-heavy phases — resume screening and recommendation generation. `ScoreAggregator` and `StatusNotifier` handle the two deterministic phases — score calculation and status notification — as plain Java objects invoked directly by the workflow, with no LLM call.

The pattern's defining property is the **explicit task dependency**: each phase writes its typed result onto `ApplicationEntity` before the next phase reads it; the workflow's step structure is the only path information travels between phases.

One governance mechanism is wired around the pipeline:

- A **CI test gate** enforces that `ScoreAggregator` and `StatusNotifier` have passing unit tests before the service is deployed or started. Because these components are pure Java with no LLM calls, their behaviour is deterministic and their correctness is verifiable offline. A failing test in either component stops the build before any candidate data can enter a running system. This is the right cut for deterministic agents in a mixed pipeline: the gate is cheap, fast, and reliable.

The blueprint shows that not every agent in a sequential pipeline needs to be an LLM. Deterministic components earn their place in the same workflow by being easier to test, audit, and gate — a property the CI gate makes explicit.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in a **candidate name** and **resume text** (or picks one of three seeded candidates — `Alice Chen (Staff Engineer)`, `Bob Martins (Product Manager)`, `Carol Wu (Data Scientist)`).
2. The user selects a **target role** from the dropdown and clicks **Submit application**. The UI POSTs to `/api/applications` and receives an `applicationId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `SCREENING` — the workflow has started `screenStep` and `ScreeningAgent` is processing the resume.
4. Within ~15–20 s the card reaches `SCREENED`. The right pane shows the `ScreenResult` — a brief fit assessment, a list of matched skills, and an `screened: true/false` flag.
5. Within ~1 s the card reaches `SCORED`. `ScoreAggregator` has run deterministically; the right pane shows five dimension scores and a total.
6. Within ~10 s the card reaches `RECOMMENDED`. `ScreeningAgent` has run its second task; the right pane shows the recommendation text (ADVANCE / HOLD / REJECT) and a one-sentence rationale.
7. Within ~1 s the card reaches `NOTIFIED`. `StatusNotifier` has recorded the `StatusUpdate`; the right pane shows the notification payload.
8. The `QualityGate` result chip appears: a green PASS or red FAIL with a one-line reason. Score outside [0,100], missing dimension, or empty recommendation text causes FAIL.
9. The user can submit another candidate; the live list keeps all history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ApplicationEndpoint` | `HttpEndpoint` | `/api/applications/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ApplicationEntity`, `ApplicationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ApplicationEntity` | `EventSourcedEntity` | Per-application lifecycle: created → screening → screened → scoring → scored → recommending → recommended → notified. Source of truth. | `ApplicationEndpoint`, `ApplicationPipelineWorkflow` | `ApplicationView` |
| `ApplicationPipelineWorkflow` | `Workflow` | One workflow per applicationId. Steps: `screenStep` → `scoreStep` → `recommendStep` → `notifyStep`. Each LLM step calls `runSingleTask` on `ScreeningAgent`; deterministic steps invoke `ScoreAggregator` or `StatusNotifier` directly. | started by `ApplicationEndpoint` after `CREATED` | `ScreeningAgent`, `ScoreAggregator`, `StatusNotifier`, `ApplicationEntity` |
| `ScreeningAgent` | `AutonomousAgent` | The single LLM agent. Declares two `Task<R>` constants in `ScreeningTasks.java`: `SCREEN_RESUME` → `ScreenResult`, `GENERATE_RECOMMENDATION` → `Recommendation`. Each task is registered with its phase-appropriate function tools. | invoked by `ApplicationPipelineWorkflow` | returns typed results |
| `ScoreAggregator` | plain Java class (no Akka primitive) | Deterministic scoring. Input: `ScreenResult`, `ApplicationRecord` metadata (role, experience years, location). Output: `CandidateScore{totalScore, dimensionScores, scoredAt}`. No LLM call. | called from `scoreStep` | returns `CandidateScore` |
| `StatusNotifier` | plain Java class (no Akka primitive) | Deterministic notification record. Input: `AggregatedScore` (score + recommendation). Output: `StatusUpdate{applicationId, decision, score, notifiedAt}`. No LLM call; no external I/O in this blueprint. | called from `notifyStep` | returns `StatusUpdate` |
| `ScoreTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `lookupRoleRequirements(role)` and `checkSkillMatch(skills, requirements)`. Reads from `src/main/resources/sample-data/roles/*.json` for deterministic offline output. | called from SCREEN task | returns `RoleRequirements` / `SkillMatchResult` |
| `RecommendTools` | function-tools class | Implements `fetchCandidateHistory(candidateId)` and `lookupMarketBenchmark(role)`. Pure in-memory reads. | called from RECOMMEND task | returns `CandidateHistory` / `MarketBenchmark` |
| `QualityGate` | plain class (no Akka primitive) | Deterministic on-decision evaluator. Inputs: `AggregatedScore`. Output: `GateResult{pass, reason, evaluatedAt}`. | called from `notifyStep` after `StatusUpdate` | returns gate result |
| `ApplicationView` | `View` | Read model: one row per application for the UI. | `ApplicationEntity` events | `ApplicationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SkillMatch(String skill, boolean matched, String evidence) {}

record ScreenResult(
    String candidateId,
    String role,
    boolean passedInitialScreen,
    List<SkillMatch> skillMatches,
    String fitSummary,
    Instant screenedAt
) {}

record DimensionScore(String dimension, int score, String rationale) {}
// dimensions: "skills-match", "experience-years", "role-seniority", "location-preference", "screen-flag"

record CandidateScore(
    int totalScore,       // 0..100
    List<DimensionScore> dimensionScores,
    Instant scoredAt
) {}

enum Decision { ADVANCE, HOLD, REJECT }

record Recommendation(
    Decision decision,
    String rationale,
    Instant recommendedAt
) {}

record AggregatedScore(
    CandidateScore score,
    Recommendation recommendation
) {}

record StatusUpdate(
    String applicationId,
    Decision decision,
    int score,
    String notificationPayload,
    Instant notifiedAt
) {}

record GateResult(
    boolean pass,
    String reason,
    Instant evaluatedAt
) {}

record ApplicationRecord(
    String applicationId,
    Optional<String> candidateName,
    Optional<String> resumeText,
    Optional<String> targetRole,
    Optional<ScreenResult> screenResult,
    Optional<CandidateScore> candidateScore,
    Optional<Recommendation> recommendation,
    Optional<AggregatedScore> aggregatedScore,
    Optional<StatusUpdate> statusUpdate,
    Optional<GateResult> gateResult,
    ApplicationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ApplicationStatus {
    CREATED, SCREENING, SCREENED, SCORING, SCORED,
    RECOMMENDING, RECOMMENDED, NOTIFYING, NOTIFIED, FAILED
}
```

Events on `ApplicationEntity`: `ApplicationCreated`, `ScreeningStarted`, `ScreeningCompleted`, `ScoringStarted`, `ScoringCompleted`, `RecommendationStarted`, `RecommendationCompleted`, `NotificationRecorded`, `GateResultRecorded`, `ApplicationFailed`.

Every nullable lifecycle field on the `ApplicationRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/applications` — body `{ candidateName, resumeText, targetRole }` → `{ applicationId }`.
- `GET /api/applications` — list all applications, newest-first.
- `GET /api/applications/{id}` — one application.
- `GET /api/applications/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Non-AI Agents (Score Aggregator)</title>`.

The App UI tab is a two-column layout: a left rail with the live list of applications (status pill + candidate name + role + age) and a right pane with the selected application's detail — screen result, dimension score table, recommendation chip, status update payload, gate result chip, and a pipeline-phase timeline showing which steps used the LLM vs. deterministic Java.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — CI test gate (build-gate)**: `ScoreAggregator` and `StatusNotifier` are pure Java classes with no external dependencies. The build system runs their unit tests (`ScoreAggregatorTest`, `StatusNotifierTest`) before packaging the service. A test failure stops the build and prevents the service from starting. The gate checks: (a) `ScoreAggregator` returns a total score in [0, 100] for every test fixture; (b) all five `DimensionScore` entries are always present; (c) `StatusNotifier` always returns a non-null `StatusUpdate.notificationPayload`; (d) `StatusNotifier.decision` matches the `Recommendation.decision` it received. This is the right governance cut for deterministic agents: the component is fully verifiable offline, and a gate at build time is cheaper and more reliable than a runtime guardrail that would need to replicate the same checks.

The `QualityGate` evaluator runs at runtime after `AggregationCompleted` and applies the same four checks to the live output, ensuring deployed behaviour matches test behaviour.

## 9. Agent prompts

- `ScreeningAgent` → `prompts/screening-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, use only the tools appropriate to that phase, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits seeded candidate `Alice Chen (Staff Engineer)` for the `Staff Software Engineer` role; within 60 s the application reaches `NOTIFIED` with a non-empty `ScreenResult`, five `DimensionScore` entries, a `Recommendation`, a `StatusUpdate`, and a `GateResult` of PASS.
2. **J2** — `ScoreAggregatorTest` and `StatusNotifierTest` pass during build; the service starts. A test is then forced to fail (mock fixture returning a score of 150); the build fails; the service does not start. This is the CI gate in action.
3. **J3** — The `QualityGate` catches a runtime anomaly: a `CandidateScore` is injected with only 3 of 5 `DimensionScore` entries (mock path). The `GateResult` is FAIL; the UI flags the card; the `StatusUpdate` is still recorded (gate is non-blocking at runtime — it annotates, not blocks).
4. **J4** — The pipeline step log shows `screenStep` and `recommendStep` as LLM-task steps; `scoreStep` and `notifyStep` as deterministic steps with no `runSingleTask` call. This is the audit evidence that the mixed-agent property holds.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named score-aggregator demonstrating the sequential-pipeline x hr-recruiting
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-hr-recruiting-mixed-deterministic-agents. Java package
io.akka.samples.nonaiagentsscoreaggregator. Akka 3.6.0. HTTP port 9696.

Components to wire (exactly):

- 1 AutonomousAgent ScreeningAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/screening-agent.md>) and two .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one for SCREEN_RESUME, one for GENERATE_RECOMMENDATION. Function tools
  ScoreTools (for SCREEN_RESUME) and RecommendTools (for GENERATE_RECOMMENDATION) are ALL
  registered on the agent via .tools(...). Phase discipline is the agent's responsibility,
  described in the system prompt; there is no before-tool-call guardrail in this blueprint.
  On task completion the agent returns the typed result.

- 1 Workflow ApplicationPipelineWorkflow per applicationId with four steps:
  * screenStep — emits ScreeningStarted on the entity, then calls componentClient
    .forAutonomousAgent(ScreeningAgent.class, "agent-" + applicationId).runSingleTask(
      TaskDef.instructions("Candidate: " + candidateName + "\nRole: " + targetRole +
        "\nResume:\n" + resumeText + "\nPhase: SCREEN\nUse lookupRoleRequirements and
        checkSkillMatch to assess the candidate.")
        .metadata("applicationId", applicationId)
        .metadata("phase", "SCREEN")
        .taskType(ScreeningTasks.SCREEN_RESUME)
    ). Reads forTask(taskId).result(SCREEN_RESUME) to get ScreenResult. Writes
    ApplicationEntity.recordScreenResult(screenResult). WorkflowSettings.stepTimeout 60s.
  * scoreStep — emits ScoringStarted, then invokes ScoreAggregator.score(screenResult,
    candidateName, targetRole, experienceYears, locationPreference) directly (no LLM call).
    Writes ApplicationEntity.recordScore(candidateScore). stepTimeout 10s.
  * recommendStep — emits RecommendationStarted, then runSingleTask with TaskDef.instructions
    (formatRecommendContext(screenResult, candidateScore, targetRole)) and
    metadata.phase = "RECOMMEND", taskType GENERATE_RECOMMENDATION. Reads result. Writes
    ApplicationEntity.recordRecommendation(recommendation). stepTimeout 60s.
  * notifyStep — emits NotificationRecorded. Invokes StatusNotifier.notify(applicationId,
    recommendation, candidateScore) directly (no LLM call). Then invokes
    QualityGate.evaluate(candidateScore) and writes both StatusUpdate and GateResult onto
    ApplicationEntity via recordStatusUpdate(statusUpdate) and
    recordGateResult(gateResult). stepTimeout 10s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ApplicationPipelineWorkflow::error). The error step writes
  ApplicationFailed and ends.

- 1 EventSourcedEntity ApplicationEntity (one per applicationId). State
  ApplicationRecord{applicationId, candidateName: Optional<String>,
  resumeText: Optional<String>, targetRole: Optional<String>,
  screenResult: Optional<ScreenResult>, candidateScore: Optional<CandidateScore>,
  recommendation: Optional<Recommendation>, aggregatedScore: Optional<AggregatedScore>,
  statusUpdate: Optional<StatusUpdate>, gateResult: Optional<GateResult>,
  status: ApplicationStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  ApplicationStatus enum: CREATED, SCREENING, SCREENED, SCORING, SCORED, RECOMMENDING,
  RECOMMENDED, NOTIFYING, NOTIFIED, FAILED.
  Events: ApplicationCreated{candidateName, resumeText, targetRole}, ScreeningStarted,
  ScreeningCompleted{screenResult}, ScoringStarted, ScoringCompleted{candidateScore},
  RecommendationStarted, RecommendationCompleted{recommendation},
  NotificationRecorded{statusUpdate}, GateResultRecorded{gateResult},
  ApplicationFailed{reason}.
  Commands: create, startScreening, recordScreenResult, startScoring, recordScore,
  startRecommendation, recordRecommendation, recordStatusUpdate, recordGateResult,
  fail, getApplication.
  emptyState() returns ApplicationRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View ApplicationView with row type ApplicationRow that mirrors ApplicationRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes ApplicationEntity
  events. ONE query getAllApplications: SELECT * AS applications FROM application_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ApplicationEndpoint at /api with POST /applications (body {candidateName, resumeText,
    targetRole}; mints applicationId; calls ApplicationEntity.create(...); then starts
    ApplicationPipelineWorkflow with id "pipeline-" + applicationId; returns {applicationId}),
    GET /applications (list from getAllApplications, sorted newest-first),
    GET /applications/{id} (one row), GET /applications/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- ScreeningTasks.java declaring two Task<R> constants:
    SCREEN_RESUME = Task.name("Screen resume").description("Assess the candidate resume
      against the target role using lookupRoleRequirements and checkSkillMatch")
      .resultConformsTo(ScreenResult.class);
    GENERATE_RECOMMENDATION = Task.name("Generate recommendation").description(
      "Produce a hiring recommendation (ADVANCE / HOLD / REJECT) with a one-sentence
      rationale, informed by the score and screening results")
      .resultConformsTo(Recommendation.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- ScoreTools.java — @FunctionTool lookupRoleRequirements(String role) -> RoleRequirements
  reading from src/main/resources/sample-data/roles/*.json keyed by role slug;
  @FunctionTool checkSkillMatch(List<String> skills, RoleRequirements requirements)
  -> SkillMatchResult returning per-skill match evidence.

- RecommendTools.java — @FunctionTool fetchCandidateHistory(String candidateId)
  -> CandidateHistory (reads from src/main/resources/sample-data/history/*.json or
  returns empty history if no file exists); @FunctionTool lookupMarketBenchmark(String role)
  -> MarketBenchmark reading the market benchmark from
  src/main/resources/sample-data/benchmarks/*.json.

- ScoreAggregator.java — pure deterministic Java. Method:
  CandidateScore score(ScreenResult screenResult, String targetRole,
    int experienceYears, String locationPreference). Five dimensions:
  "skills-match" (0-30 based on matched skills ratio from screenResult.skillMatches),
  "experience-years" (0-25 based on experienceYears vs. role requirement),
  "role-seniority" (0-20 based on seniority match),
  "location-preference" (0-15 based on location preference match),
  "screen-flag" (0 or 10 based on screenResult.passedInitialScreen).
  totalScore = sum of dimension scores, capped at 100.
  Must have unit test ScoreAggregatorTest (CI gate H1).

- StatusNotifier.java — pure deterministic Java. Method:
  StatusUpdate notify(String applicationId, Recommendation recommendation,
    CandidateScore score). Builds a JSON-formatted notificationPayload string with the
  decision, score, and a timestamp. No external I/O. Returns StatusUpdate.
  Must have unit test StatusNotifierTest (CI gate H1).

- QualityGate.java — pure deterministic logic (no LLM). Input: CandidateScore.
  Output: GateResult with pass and reason. Four checks:
  (1) totalScore in [0, 100],
  (2) all five expected dimension keys present in dimensionScores,
  (3) each DimensionScore.score in [0, its maximum],
  (4) at least one DimensionScore has score > 0 (not all-zero pathological case).
  pass = all four checks pass. reason names the first failing check, or "all checks passed".

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9696 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/applications.jsonl with 5 seeded candidate lines covering
  the three seeded candidates named in the user flows plus two extras.

- src/main/resources/sample-data/roles/*.json — three role files (staff-software-engineer,
  product-manager, data-scientist) each carrying required skills, experience floor,
  seniority level, and location preferences.

- src/main/resources/sample-data/history/*.json — candidate history files for the three
  seeded candidates (optional; if absent, RecommendTools.fetchCandidateHistory returns empty).

- src/main/resources/sample-data/benchmarks/*.json — market benchmark files for the three
  role slugs (salary range, demand level, supply level).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (H1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = true (candidateName
  and resumeText are PII), decisions.authority_level = recommend-only (the pipeline
  produces a recommendation; a human recruiter decides), oversight.human_in_loop = true
  (a recruiter reviews before acting), operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "score-out-of-range", "missing-dimension",
  "hallucinated-recommendation", "resume-bias", "gate-failure"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/screening-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Non-AI Agents (Score Aggregator)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of application cards; right = selected-application detail with candidate name,
  role, screen result, dimension score table, recommendation chip, status update payload,
  gate result chip, and pipeline-phase timeline showing LLM vs. deterministic steps).
  Browser title exactly:
  <title>Akka Sample: Non-AI Agents (Score Aggregator)</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(applicationId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    screen-resume.json — 6 ScreenResult entries, each paired with one seeded candidate.
      Each entry's tool_calls array contains 1 lookupRoleRequirements(role) + 1
      checkSkillMatch(skills, requirements) call. Plus 1 entry with
      passedInitialScreen = false (the weak-candidate path) so J3's gate anomaly
      path is reachable.
    generate-recommendation.json — 6 Recommendation entries paired one-to-one with the
      screen entries. Decisions are ADVANCE (×3), HOLD (×2), REJECT (×1). tool_calls
      contain fetchCandidateHistory + lookupMarketBenchmark in order.
- A MockModelProvider.seedFor(applicationId) helper makes per-application selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ScreeningAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ScreeningTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (screenStep
  60s, scoreStep 10s, recommendStep 60s, notifyStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the ApplicationRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ScreeningTasks.java with SCREEN_RESUME, GENERATE_RECOMMENDATION constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9696 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The mixed-agent invariant: ScreeningAgent is the ONLY AutonomousAgent (one LLM-calling
  component). ScoreAggregator and StatusNotifier are pure Java objects invoked directly by
  the workflow — they are never wrapped as AutonomousAgents. QualityGate is similarly a
  pure Java class.
- The sequential-pipeline invariant: each phase's typed output is written to
  ApplicationEntity before the next phase runs. The workflow step structure is the only
  path information travels between phases.
- Task dependency is carried by typed task results: screenStep writes ScreenResult onto the
  entity; scoreStep reads it (deterministic); recommendStep reads both (LLM); notifyStep
  reads both (deterministic). The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
