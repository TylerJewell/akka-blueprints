# SPEC — hr-shortlister

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** HR Candidate Shortlister.
**One-line pitch:** A recruiter submits a resume; one `ShortlistAgent` walks it through three task phases — **PARSE** raw applicant data, **SCORE** the structured profile against job criteria, **DECIDE** shortlist / hold / reject — with protected attributes redacted before scoring, a periodic fairness monitor on outcomes, and a human approval gate before the ERPNext write.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an HR-recruiting domain. One `ShortlistAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PARSE task's typed output (`Profile`) becomes the SCORE task's instruction context; the SCORE task's typed output (`CandidateScore`) becomes the DECIDE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` sanitizer** sits between the agent and every SCORE-phase tool call. It inspects the in-flight `Profile` for special-category fields — `age`, `gender`, `nationality`, `disabilityIndicator` — and replaces their values with `"[REDACTED]"` before the tool body executes. The original values are never written to the scored record; redaction is the default, not an option. A `SpecialCategoryRedacted` event records each redaction for audit.
- A **`eval-periodic` fairness drift monitor** (`FairnessScorer`) runs outside the per-application pipeline, triggered by a scheduled workflow that fires every 24 hours. It reads the last 30 days of `ApplicationRow` records from `ApplicationView`, buckets shortlisting outcomes by gender cohort and nationality cohort, and computes the shortlisting-rate ratio for each cohort pair. When any ratio exceeds the configured threshold (default 1.20), the scorer emits a `DriftReport` visible in the Eval Matrix tab.
- A **`hitl` approval gate** (`approvalStep` in `ShortlistingWorkflow`) suspends the workflow after `ShortlistDecided` is emitted, waiting for a recruiter to call `POST /api/applications/{id}/decision` with `approve` or `override`. Only after approval does the workflow call `ErpNextAdapter.writeApplicant(...)` and emit `WrittenToErpNext`. An override replaces the agent's decision with the recruiter's decision before the ERPNext write.

The blueprint shows that a sequential pipeline in HR is not just a chain of LLM calls — the task-boundary handoffs enforce the dependency contract, the sanitizer enforces the protected-attribute boundary, the fairness monitor detects population-level drift that no single-application guardrail can see, and the approval gate enforces the human-authority boundary before any external write.

## 3. User-facing flows

The recruiter opens the App UI tab.

1. The recruiter pastes a resume (or picks one of three seeded applicant profiles — `Software Engineer L3 candidate`, `Product Manager mid-level candidate`, `Data Analyst entry-level candidate`) and enters a job ID.
2. The recruiter clicks **Submit application**. The UI POSTs to `/api/applications` and receives an `applicationId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the agent has been handed the PARSE task.
4. Within ~10–20 s the card reaches `SCORED`. The structured `Profile` is visible in the card detail (name, extracted skills, experience years, education). Any redacted fields are shown as `[REDACTED]` chips. The `CandidateScore` is visible (overall score, per-criterion breakdown).
5. Within ~10–20 s more the card reaches `AWAITING_APPROVAL`. The agent's `ShortlistDecision` (`SHORTLIST` / `HOLD` / `REJECT` with rationale) is shown. A yellow banner prompts the recruiter to approve or override.
6. The recruiter clicks **Approve** (or selects an override decision). The application moves to `APPROVED` and within ~5 s to `WRITTEN` — the ERPNext adapter stub has been called and `WrittenToErpNext` recorded.
7. The recruiter can submit another resume; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ShortlistEndpoint` | `HttpEndpoint` | `/api/applications/*` — submit, list, get, decision, SSE; serves `/api/metadata/*`. | — | `ApplicationEntity`, `ApplicationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ApplicationEntity` | `EventSourcedEntity` | Per-application lifecycle: received → parsing → parsed → scoring → scored → deciding → decided → awaiting-approval → approved → written. Source of truth. | `ShortlistEndpoint`, `ShortlistingWorkflow` | `ApplicationView` |
| `ShortlistingWorkflow` | `Workflow` | One workflow per applicationId. Steps: `parseStep` → `scoreStep` → `shortlistStep` → `approvalStep` → `erpNextStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `approvalStep` suspends waiting for recruiter decision. | started by `ShortlistEndpoint` after `RECEIVED` | `ShortlistAgent`, `ApplicationEntity`, `ErpNextAdapter` |
| `ShortlistAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `ShortlistTasks.java`: `PARSE_RESUME` → `Profile`, `SCORE_PROFILE` → `CandidateScore`, `DECIDE_SHORTLIST` → `ShortlistDecision`. Each task is registered with the phase-appropriate function tools. | invoked by `ShortlistingWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `extractProfile(resumeText)` and `normalizeEducation(rawEducation)`. Reads seeded profiles from `src/main/resources/sample-data/profiles/*.json` for deterministic offline output. | called from PARSE task | returns `Profile` fields |
| `ScoreTools` | function-tools class | Implements `evaluateCriteria(profile, jobCriteria)` and `computeOverallScore(criteriaScores)`. Pure in-memory scoring. Receives a sanitized `Profile` (special-category fields already `[REDACTED]`). | called from SCORE task | returns `List<CriterionScore>` / `OverallScore` |
| `ShortlistTools` | function-tools class | Implements `classifyDecision(score, threshold)` and `generateRationale(score, criteria)`. | called from DECIDE task | returns `Decision` / `String` |
| `SpecialCategoryGuardrail` | `before-tool-call` guardrail (registered on `ShortlistAgent`) | Before any SCORE-phase tool call, inspects the current in-flight `Profile` (looked up from `ApplicationEntity` via `applicationId` in `TaskDef.metadata`), replaces special-category field values with `"[REDACTED]"`, calls `ApplicationEntity.recordRedaction(...)` per field, then passes the sanitized profile. Never rejects — always passes, but always sanitizes. | every SCORE-phase tool call | sanitize + pass |
| `FairnessScorer` | plain class (no Akka primitive) | Periodic fairness evaluator. Inputs: list of `ApplicationRow` records for a 30-day window. Output: `DriftReport{cohort, metric, ratio, threshold, breached}`. | called from `FairnessDriftWorkflow` (scheduled) | returns `DriftReport` |
| `FairnessDriftWorkflow` | `Workflow` | Scheduled workflow that fires daily. Reads `ApplicationView`, passes rows to `FairnessScorer`, writes `DriftReport` to `DriftReportEntity`. | timer / scheduler | `ApplicationView`, `FairnessScorer`, `DriftReportEntity` |
| `DriftReportEntity` | `EventSourcedEntity` | Holds the latest `DriftReport` per cohort pair. | `FairnessDriftWorkflow` | `ShortlistEndpoint` (read) |
| `ErpNextAdapter` | plain class | HTTP client stub for ERPNext `/api/resource/Job Applicant`. Returns deterministic response in dev mode. | `ShortlistingWorkflow.erpNextStep` | external ERPNext (or stub) |
| `ApplicationView` | `View` | Read model: one row per application for the UI and fairness scanner. | `ApplicationEntity` events | `ShortlistEndpoint`, `FairnessDriftWorkflow` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Profile(
    String applicantName,
    String email,
    String resumeText,
    List<String> skills,
    int experienceYears,
    String highestEducation,
    String currentTitle,
    // special-category fields — always redacted before SCORE phase
    String age,              // "[REDACTED]" after sanitization
    String gender,           // "[REDACTED]" after sanitization
    String nationality,      // "[REDACTED]" after sanitization
    String disabilityIndicator // "[REDACTED]" after sanitization
) {}

record CriterionScore(String criterionId, String criterionName, int score, String justification) {}

record CandidateScore(
    List<CriterionScore> criterionScores,
    int overallScore,   // 0..100
    Instant scoredAt
) {}

record ShortlistDecision(
    Decision decision,         // SHORTLIST / HOLD / REJECT
    String rationale,
    int confidenceScore,       // 0..100
    Instant decidedAt
) {}

enum Decision { SHORTLIST, HOLD, REJECT }

record RecruiterDecision(
    String recruiterId,
    Decision decision,
    String overrideRationale,  // null if approving agent decision as-is
    Instant decidedAt
) {}

record RedactionRecord(String fieldName, String originalPresence, Instant redactedAt) {}

record ApplicationRecord(
    String applicationId,
    String jobId,
    Optional<Profile> profile,
    Optional<CandidateScore> score,
    Optional<ShortlistDecision> agentDecision,
    Optional<RecruiterDecision> recruiterDecision,
    ApplicationStatus status,
    List<RedactionRecord> redactions,
    Instant receivedAt,
    Optional<Instant> finishedAt
) {}

enum ApplicationStatus {
    RECEIVED, PARSING, PARSED, SCORING, SCORED, DECIDING, DECIDED,
    AWAITING_APPROVAL, APPROVED, WRITTEN, FAILED
}

record DriftReport(
    String cohortDimension,   // "gender" or "nationality"
    String cohortA,
    String cohortB,
    double cohortARate,
    double cohortBRate,
    double ratio,
    double threshold,
    boolean breached,
    Instant computedAt
) {}
```

Events on `ApplicationEntity`: `ResumeReceived`, `ParseStarted`, `ProfileParsed`, `ScoreStarted`, `ScoreAssigned`, `DecisionStarted`, `ShortlistDecided`, `SpecialCategoryRedacted`, `ApprovalRequested`, `RecruiterApproved`, `RecruiterOverridden`, `WrittenToErpNext`, `ApplicationFailed`.

Every nullable lifecycle field on the `ApplicationRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/applications` — body `{ jobId, resumeText }` → `{ applicationId }`.
- `GET /api/applications` — list all applications, newest-first.
- `GET /api/applications/{id}` — one application.
- `POST /api/applications/{id}/decision` — body `{ recruiterId, decision, overrideRationale? }` → `204`.
- `GET /api/applications/sse` — Server-Sent Events; one event per state transition.
- `GET /api/drift-reports` — list of `DriftReport` records, newest-first.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: HR Candidate Shortlister</title>`.

The App UI tab is a two-column layout: a left rail with the live list of applications (status pill + applicant name + age) and a right pane with the selected application's detail — profile (with redaction chips on special-category fields), score breakdown, agent decision with rationale, recruiter decision panel (approve / override buttons when `AWAITING_APPROVAL`), and a redaction-log strip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — `before-tool-call` sanitizer (special-category redaction)**: `SpecialCategoryGuardrail` is registered on `ShortlistAgent` and runs before every SCORE-phase tool call. It reads the current `ApplicationEntity` state for the application (looked up via `applicationId` in the in-flight `TaskDef.metadata`), identifies populated special-category fields (`age`, `gender`, `nationality`, `disabilityIndicator`), and replaces their values with `"[REDACTED]"` in the profile object passed to the tool. For each redacted field it calls `ApplicationEntity.recordRedaction(fieldName, ...)` so the redaction is visible in the UI's redaction-log strip and in the audit log. The tool body executes with the sanitized profile — the original values never reach the scoring logic.
- **H1 — `hitl` approval gate**: `approvalStep` in `ShortlistingWorkflow` emits `ApprovalRequested` and then suspends via `effects().pause()`. The workflow resumes only when `ShortlistEndpoint` receives a `POST /api/applications/{id}/decision` call and forwards it to the workflow. If the recruiter approves, the agent's decision is preserved and `RecruiterApproved` is emitted. If the recruiter overrides, the override decision replaces the agent's decision and `RecruiterOverridden` is emitted with the override rationale. The workflow then calls `ErpNextAdapter` and emits `WrittenToErpNext`.
- **E1 — `eval-periodic` fairness drift monitor**: `FairnessDriftWorkflow` fires daily (configurable). `FairnessScorer` reads all `ApplicationRow` records from the last 30 days via `ApplicationView.getApplicationsInWindow(from, to)`. For each cohort dimension (`gender`, `nationality`) it computes the shortlisting rate per cohort value and the ratio between the highest- and lowest-rate cohorts. When the ratio exceeds `threshold` (default 1.20), a `DriftReport` with `breached = true` is written to `DriftReportEntity`. The UI's Eval Matrix tab shows the current `DriftReport` per dimension.

## 9. Agent prompts

- `ShortlistAgent` → `prompts/shortlist-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. It is explicitly instructed that scoring must treat special-category field values as `"[REDACTED]"` if present — it must not infer protected attributes from other profile fields.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Recruiter submits the seeded `Software Engineer L3 candidate` profile; within 60 s the application reaches `AWAITING_APPROVAL` with a non-empty `Profile`, a `CandidateScore` with ≥ 3 criterion scores, and a `ShortlistDecision`. Recruiter approves; within 5 s the application reaches `WRITTEN`.
2. **J2** — A resume with an explicit `"age": "34"` field is submitted. Before `evaluateCriteria` executes, `SpecialCategoryGuardrail` replaces `age` with `"[REDACTED]"`. A `SpecialCategoryRedacted` event lands on the entity. The UI's redaction-log strip shows the one redacted field. The application completes correctly.
3. **J3** — Recruiter overrides the agent's `REJECT` decision with `SHORTLIST` and supplies an `overrideRationale`. The `RecruiterOverridden` event lands; the ERPNext adapter stub is called with the override decision. The UI shows the override badge on the card.
4. **J4** — After 30+ decisions in the rolling window, `FairnessDriftWorkflow` runs. If the seeded data includes a skewed cohort, the `DriftReport` shows `breached = true` with the ratio. The Eval Matrix tab displays the report.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hr-shortlister demonstrating the sequential-pipeline x hr-recruiting cell.
Runs out of the box (no external services required; ERPNext is stubbed). Maven group io.akka.samples.
Maven artifact sequential-pipeline-hr-recruiting-hr-shortlister.
Java package io.akka.samples.aipoweredcandidateshortlistingautomationforerpnext.
Akka 3.6.0. HTTP port 9979.

Components to wire (exactly):

- 1 AutonomousAgent ShortlistAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/shortlist-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  PARSE, SCORE, and DECIDE tool sets are ALL registered on the agent; phase gating is the
  job of SpecialCategoryGuardrail for SCORE-phase redaction, NOT of conditional .tools(...)
  wiring. The before-tool-call guardrail (SpecialCategoryGuardrail) is registered on the
  agent via the agent's guardrail-configuration block.

- 1 Workflow ShortlistingWorkflow per applicationId with five steps:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(ShortlistAgent.class, "agent-" + applicationId).runSingleTask(
      TaskDef.instructions("Resume text: " + resumeText + "\nPhase: PARSE\nExtract a
      structured Profile from this resume.")
        .metadata("applicationId", applicationId)
        .metadata("phase", "PARSE")
        .taskType(ShortlistTasks.PARSE_RESUME)
    ). Reads forTask(taskId).result(PARSE_RESUME) to get Profile. Writes
    ApplicationEntity.recordProfile(profile). WorkflowSettings.stepTimeout 60s.
  * scoreStep — emits ScoreStarted, then runSingleTask with TaskDef.instructions
    (formatScoreContext(profile, jobId)) and metadata.phase = "SCORE", taskType
    SCORE_PROFILE. Writes ApplicationEntity.recordScore(score). stepTimeout 60s.
  * shortlistStep — emits DecisionStarted, then runSingleTask with TaskDef.instructions
    (formatDecisionContext(score, profile, jobId)) and metadata.phase = "DECIDE", taskType
    DECIDE_SHORTLIST. Writes ApplicationEntity.recordDecision(decision). stepTimeout 60s.
  * approvalStep — emits ApprovalRequested and suspends via effects().pause(). Resumes
    when ShortlistEndpoint forwards a recruiter decision via resumeWorkflow. stepTimeout
    set to the workflow-level max (no hard timeout — approval is async). On resume, emits
    RecruiterApproved or RecruiterOverridden.
  * erpNextStep — calls ErpNextAdapter.writeApplicant(applicationId, finalDecision,
    profile, score) and writes WrittenToErpNext. stepTimeout 30s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ShortlistingWorkflow::error). The error step writes
  ApplicationFailed and ends.

- 1 EventSourcedEntity ApplicationEntity (one per applicationId). State ApplicationRecord
  {applicationId, jobId: Optional<String>, profile: Optional<Profile>,
  score: Optional<CandidateScore>, agentDecision: Optional<ShortlistDecision>,
  recruiterDecision: Optional<RecruiterDecision>, status: ApplicationStatus,
  redactions: List<RedactionRecord>, receivedAt: Instant,
  finishedAt: Optional<Instant>}. ApplicationStatus enum: RECEIVED, PARSING, PARSED,
  SCORING, SCORED, DECIDING, DECIDED, AWAITING_APPROVAL, APPROVED, WRITTEN, FAILED.
  Events: ResumeReceived{jobId, resumeText}, ParseStarted, ProfileParsed{profile},
  ScoreStarted, ScoreAssigned{score}, DecisionStarted, ShortlistDecided{agentDecision},
  SpecialCategoryRedacted{fieldName, redactedAt}, ApprovalRequested,
  RecruiterApproved{recruiterDecision}, RecruiterOverridden{recruiterDecision},
  WrittenToErpNext{erpNextId}, ApplicationFailed{reason}.
  Commands: create, startParse, recordProfile, startScore, recordScore, startDecision,
  recordDecision, recordRedaction, requestApproval, recordRecruiterDecision,
  recordErpNextWrite, fail, getApplication. emptyState() returns
  ApplicationRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3).

- 1 EventSourcedEntity DriftReportEntity (one per cohortDimension). State: latest
  DriftReport per dimension. Events: DriftReportUpdated{report}. Commands: update, get.

- 1 Workflow FairnessDriftWorkflow — scheduled, fires daily. Step: scanStep reads
  ApplicationView.getApplicationsInWindow(from, to), calls FairnessScorer.compute(rows),
  writes result to DriftReportEntity for each cohort dimension. stepTimeout 30s.

- 1 View ApplicationView with row type ApplicationRow that mirrors ApplicationRecord
  exactly (all Optional<T> lifecycle fields preserved). Table updater consumes
  ApplicationEntity events. ONE query getAllApplications: SELECT * AS applications FROM
  application_view. No WHERE status filter (Lesson 2); caller filters client-side.
  Additional query getApplicationsInWindow(Instant from, Instant to): used by
  FairnessDriftWorkflow only.

- 2 HttpEndpoints:
  * ShortlistEndpoint at /api with POST /applications (body {jobId, resumeText}; mints
    applicationId; calls ApplicationEntity.create(jobId, resumeText); then starts
    ShortlistingWorkflow with id "shortlist-" + applicationId; returns {applicationId}),
    GET /applications (list newest-first), GET /applications/{id} (one row),
    POST /applications/{id}/decision (body {recruiterId, decision, overrideRationale?};
    resumes ShortlistingWorkflow), GET /applications/sse (SSE from view stream-updates),
    GET /drift-reports (list of DriftReport from DriftReportEntity), and three
    /api/metadata/* endpoints.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- ShortlistTasks.java declaring three Task<R> constants:
    PARSE_RESUME = Task.name("Parse resume").description("Extract a structured Profile
      from the resume text using extractProfile and normalizeEducation")
      .resultConformsTo(Profile.class);
    SCORE_PROFILE = Task.name("Score profile").description("Evaluate the sanitized Profile
      against job criteria using evaluateCriteria and computeOverallScore")
      .resultConformsTo(CandidateScore.class);
    DECIDE_SHORTLIST = Task.name("Decide shortlist").description("Classify the application
      as SHORTLIST, HOLD, or REJECT using classifyDecision and generateRationale")
      .resultConformsTo(ShortlistDecision.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {PARSE, SCORE, DECIDE}.

- ParseTools.java — @FunctionTool extractProfile(String resumeText) -> Profile reading from
  src/main/resources/sample-data/profiles/*.json keyed by applicant name slug; @FunctionTool
  normalizeEducation(String rawEducation) -> String (deterministic normalization).

- ScoreTools.java — @FunctionTool evaluateCriteria(Profile profile, String jobId) ->
  List<CriterionScore> (one CriterionScore per criterion in the job's criteria config);
  @FunctionTool computeOverallScore(List<CriterionScore>) -> int (weighted average,
  0..100).

- ShortlistTools.java — @FunctionTool classifyDecision(int overallScore, String jobId) ->
  Decision (SHORTLIST if >= threshold, HOLD if within buffer, REJECT otherwise); @
  FunctionTool generateRationale(CandidateScore score, String jobId) -> String (1-2
  sentence rationale).

- SpecialCategoryGuardrail.java — implements the before-tool-call hook. Fires only for
  SCORE-phase tool calls (Phase.SCORE). Reads the current ApplicationEntity profile by
  applicationId (carried in the TaskDef metadata). For each of {age, gender, nationality,
  disabilityIndicator}: if the value is non-null and not already "[REDACTED]", replaces
  it with "[REDACTED]" in the profile passed to the tool, and calls
  ApplicationEntity.recordRedaction(fieldName, Instant.now()). Always returns
  Guardrail.accept(sanitizedCallContext) — the guardrail sanitizes, never blocks.

- FairnessScorer.java — pure deterministic logic. Inputs: List<ApplicationRow>, from
  Instant, to Instant. Groups rows by gender value (ignoring "[REDACTED]") and by
  nationality value (ignoring "[REDACTED]"), computes shortlisting rate per cohort,
  finds the max-to-min ratio. Emits DriftReport{cohortDimension, cohortA, cohortB,
  cohortARate, cohortBRate, ratio, threshold=1.20, breached=(ratio>threshold),
  computedAt}. Returns one DriftReport per dimension.

- ErpNextAdapter.java — HTTP client stub. In dev mode (akka.javasdk.dev-mode active),
  logs the write and returns {"id": "APP-" + applicationId, "status": "Submitted"}.
  In production mode, reads ERPNEXT_BASE_URL + ERPNEXT_API_KEY from env vars and calls
  POST /api/resource/Job Applicant.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9979 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/profiles/*.json — three files, one per seeded applicant,
  each carrying a full resume text blob plus pre-extracted Profile fields (for
  ParseTools.extractProfile to return deterministically), including one profile with
  explicit special-category fields (age, gender, nationality) to exercise the sanitizer.

- src/main/resources/sample-data/jobs/*.json — three job definitions (Software Engineer L3,
  Product Manager mid-level, Data Analyst entry-level), each carrying a list of named
  criteria with weights and scoring thresholds.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md — copies for
  the metadata endpoint.

- eval-matrix.yaml at the project root with 3 controls (S1, H1, E1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with sector = hr-recruiting,
  data.data_classes.pii = true (name, email, resume text are PII),
  decisions.authority_level = gated-human (recruiter approves every shortlist decision),
  oversight.human_in_loop = true, operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "protected-attribute-leakage", "biased-shortlisting",
  "misordered-phase-tool-call", "erpnext-write-failure", "stale-recruiter-queue".

- prompts/shortlist-agent.md loaded as the agent system prompt.

- README.md at the project root (already in the blueprint folder).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs. App UI tab uses a two-column layout (left = live list of application
  cards; right = selected-application detail with profile panel, score breakdown, agent
  decision, recruiter decision panel, redaction-log strip). Browser title exactly:
  <title>Akka Sample: HR Candidate Shortlister</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ShortlistAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ShortlistTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, scoreStep 60s, shortlistStep 60s, erpNextStep 30s, error 5s). approvalStep has no
  hard timeout (human async).
- Lesson 6: every nullable lifecycle field on ApplicationRecord is Optional<T>.
- Lesson 7: ShortlistTasks.java with PARSE_RESUME, SCORE_PROFILE, DECIDE_SHORTLIST
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9979 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ShortlistAgent). The
  fairness scorer is rule-based (FairnessScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent,
  but SpecialCategoryGuardrail is the runtime mechanism that sanitizes SCORE-phase inputs.
  Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: parseStep writes Profile onto the
  entity, scoreStep reads it and builds the SCORE task's instruction context from it,
  shortlistStep reads the CandidateScore. The agent itself is stateless across phases.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
