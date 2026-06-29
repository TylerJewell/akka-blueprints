# SPEC — sequential-cv-tailor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Sequential CV Tailor.
**One-line pitch:** A recruiter submits a candidate profile and a job posting; one `CvGeneratorAgent` expands the profile into a structured `BaseCv`, then one `CvTailorAgent` adapts it for the posting — candidate PII sanitized out of every model payload before it leaves the workflow.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an HR recruiting domain. Two `AutonomousAgent` components each run one typed task. The pattern's defining property is the **explicit task dependency**: `CvGeneratorAgent`'s typed output (`BaseCv`) becomes `CvTailorAgent`'s instruction context. The tailor agent never sees the raw `CandidateProfile` — the workflow's typed handoff is the only path candidate data travels between stages.

One governance mechanism is wired around the pipeline:

- A **`before-call` sanitizer** registered on both agents strips PII fields — `candidateName`, `email`, `phone`, `address`, and `dateOfBirth` — from every payload presented to the model. The sanitizer runs before each LLM call; the stripped payload is what the model receives. A `SanitizationApplied` event is emitted for every call where at least one field was removed, providing an audit trail without logging the suppressed values.

The blueprint shows that PII governance in a sequential pipeline is not a single gate on input ingestion — it must be applied at every model boundary, because the intermediate typed result (`BaseCv`) may re-surface personal fields the generator was asked to include.

## 3. User-facing flows

The user opens the App UI tab.

1. The recruiter selects a **candidate** from the seeded fixture list (or pastes a raw profile text) and selects a **job posting** from the seeded fixture list (or pastes a job description).
2. The recruiter clicks **Tailor CV**. The UI POSTs to `/api/cv-requests` and receives a `requestId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `GENERATING` — the workflow has started `generateStep` and `CvGeneratorAgent` is running its `GENERATE_BASE_CV` task.
4. Within ~15–25 s the card reaches `GENERATED`. The right pane shows the `BaseCv` detail: summary, skills list, and experience entries. `CvGeneratorAgent`'s task returned; the workflow recorded `BaseCvReady` and started `tailorStep`.
5. Within ~15–25 s more the card reaches `TAILORED`. The right pane shows the `TailoredCv` with the rewritten summary, highlighted keyword matches, and modified experience bullets.
6. Within ~2 s the card reaches `SCORED`. The right pane shows a 1–5 alignment score chip and a one-line rationale explaining which required keywords were matched or missed.
7. The recruiter can submit another request; the live list keeps the history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CvEndpoint` | `HttpEndpoint` | `/api/cv-requests/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `CvRequestEntity`, `CvRequestView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CvRequestEntity` | `EventSourcedEntity` | Per-request lifecycle: created → generating → generated → tailoring → tailored → scored. Source of truth. | `CvEndpoint`, `CvPipelineWorkflow` | `CvRequestView` |
| `CvPipelineWorkflow` | `Workflow` | One workflow per requestId. Steps: `generateStep` → `tailorStep` → `evalStep`. Each step runs one task on one agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `CvEndpoint` after `CREATED` | `CvGeneratorAgent`, `CvTailorAgent`, `CvRequestEntity` |
| `CvGeneratorAgent` | `AutonomousAgent` | Declares one `Task<BaseCv>` constant in `CvTasks.java`: `GENERATE_BASE_CV`. Given a `CandidateProfile`, calls `ProfileTools` to expand and structure it. | invoked by `CvPipelineWorkflow.generateStep` | returns `BaseCv` |
| `CvTailorAgent` | `AutonomousAgent` | Declares one `Task<TailoredCv>` constant in `CvTasks.java`: `TAILOR_CV`. Given a `BaseCv` and `JobPosting`, calls `TailorTools` to adapt the CV. | invoked by `CvPipelineWorkflow.tailorStep` | returns `TailoredCv` |
| `ProfileTools` | function-tools class (POJO with `@FunctionTool` methods) | `expandSummary(profile)`, `extractSkills(profile)`, `formatExperience(profile)`. Reads from `src/main/resources/sample-data/candidates/*.json`. | called from `GENERATE_BASE_CV` task | returns `String` / `List<Skill>` / `List<ExperienceEntry>` |
| `TailorTools` | function-tools class | `matchKeywords(baseCv, posting)`, `rewriteSummary(baseCv, matchedKeywords, posting)`, `adjustExperience(baseCv, posting)`. | called from `TAILOR_CV` task | returns `List<KeywordMatch>` / `String` / `List<ExperienceEntry>` |
| `PiiSanitizer` | `before-call` sanitizer (registered on both agents) | Strips `candidateName`, `email`, `phone`, `address`, `dateOfBirth` from the payload before each model call. Emits `SanitizationApplied` on the entity when any field is removed. | every model call on both agents | scrubbed payload |
| `AlignmentScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `TailoredCv`, `JobPosting`. Output: `AlignmentResult{score, rationale, keywordsCovered, keywordsMissed}`. | called from `evalStep` | returns score |
| `CvRequestView` | `View` | Read model: one row per request for the UI. | `CvRequestEntity` events | `CvEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CandidateProfile(
    String profileId,
    String candidateName,
    String email,
    String phone,
    Optional<String> address,
    Optional<String> dateOfBirth,
    String rawSummary,
    List<String> rawSkills,
    List<RawExperience> rawExperience
) {}

record RawExperience(String company, String title, String period, String description) {}

record Skill(String name, String level) {}   // level: beginner | intermediate | expert

record ExperienceEntry(String company, String title, String period, String bulletPoints) {}

record BaseCv(
    String generatedSummary,
    List<Skill> skills,
    List<ExperienceEntry> experience,
    Instant generatedAt
) {}

record JobPosting(
    String postingId,
    String title,
    String company,
    String description,
    List<String> requiredKeywords,
    List<String> preferredKeywords
) {}

record KeywordMatch(String keyword, boolean required, String sourceSection) {}

record TailoredCv(
    String tailoredSummary,
    List<Skill> skills,
    List<ExperienceEntry> experience,
    List<KeywordMatch> keywordMatches,
    Instant tailoredAt
) {}

record AlignmentResult(
    int score,            // 1..5
    String rationale,
    List<String> keywordsCovered,
    List<String> keywordsMissed,
    Instant scoredAt
) {}

record CvRequestRecord(
    String requestId,
    Optional<String> candidateProfileId,
    Optional<String> jobPostingId,
    Optional<BaseCv> baseCv,
    Optional<TailoredCv> tailoredCv,
    Optional<AlignmentResult> alignmentResult,
    int sanitizationCount,
    CvRequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CvRequestStatus {
    CREATED, GENERATING, GENERATED, TAILORING, TAILORED, SCORED, FAILED
}
```

Events on `CvRequestEntity`: `CvRequestCreated`, `GenerationStarted`, `BaseCvReady`, `TailoringStarted`, `TailoredCvReady`, `QualityScored`, `SanitizationApplied`, `RequestFailed`.

Every nullable lifecycle field on the `CvRequestRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/cv-requests` — body `{ candidateProfileId, jobPostingId }` → `{ requestId }`.
- `GET /api/cv-requests` — list all requests, newest-first.
- `GET /api/cv-requests/{id}` — one request record.
- `GET /api/cv-requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Sequential CV Tailor</title>`.

The App UI tab is a two-column layout: a left rail with the live list of CV requests (status pill + candidate + posting + age) and a right pane with the selected request's detail — candidate and posting headers, `BaseCv` section, `TailoredCv` section with keyword-match chips, alignment score chip, and a sanitization-audit strip showing how many PII fields were stripped.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — `before-call` PII sanitizer**: `PiiSanitizer` is registered on both `CvGeneratorAgent` and `CvTailorAgent` and runs before every model call. It inspects the payload for the fields `candidateName`, `email`, `phone`, `address`, and `dateOfBirth` (checked both in the top-level call object and in any nested `CandidateProfile` reference), replaces each present field with the placeholder `[REDACTED]`, counts the fields removed, and passes the scrubbed payload to the model. If `removedCount > 0`, the sanitizer calls `CvRequestEntity.recordSanitization(requestId, removedCount, callingAgent)` to write a `SanitizationApplied{requestId, agent, removedCount, appliedAt}` event; this provides an audit trail of every model call where personal data would otherwise have been present. The sanitizer never logs the original field values.
- **E1 — `on-decision-eval`**: runs immediately after `TailoredCvReady` lands, as `evalStep` inside the workflow. `AlignmentScorer` is deterministic and rule-based (no LLM call — keeping the two-agent pipeline honest). It checks: (1) required-keyword coverage — every `JobPosting.requiredKeywords` entry appears in the `TailoredCv`'s summary + experience text (case-insensitive); (2) preferred-keyword coverage — at least 50 % of `JobPosting.preferredKeywords` appear; (3) experience-entry count ≥ 1; (4) summary non-empty. Score 1 (base) + 1 per rule satisfied. Emits `QualityScored{score:1..5, rationale, keywordsCovered, keywordsMissed}`. CVs scoring ≤ 2 are flagged on the UI card.

## 9. Agent prompts

- `CvGeneratorAgent` → `prompts/cv-generator-agent.md`. Structured-profile builder. Given a `CandidateProfile`, expand the raw summary, cluster raw skills with level inference, and structure each raw experience entry into bullet points.
- `CvTailorAgent` → `prompts/cv-tailor-agent.md`. Job-posting adapter. Given a `BaseCv` and a `JobPosting`, rewrite the summary, reorder and adjust experience bullets to surface required and preferred keywords, and return the full `TailoredCv`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Recruiter submits the seeded candidate `Alice Chen` against the seeded posting `Senior Java Engineer`; within 60 s the request reaches `SCORED` with a non-empty `BaseCv`, a non-empty `TailoredCv`, and an alignment score chip ≥ 3.
2. **J2** — The service log for any request contains no `candidateName`, `email`, `phone`, or `address` string in any model-call line; the `SanitizationApplied` event count on the entity equals the number of model calls made.
3. **J3** — A job posting whose required keywords are absent from the candidate's raw skills and experience produces a `TailoredCv` with score 1 or 2 and a rationale naming the missing keywords; the UI card border highlights red.
4. **J4** — `CvTailorAgent` receives only the `BaseCv` and `JobPosting` in its task instructions; the raw `CandidateProfile` (including the unseeded `email` and `phone` fields) does not appear in the tailor task's log entry. The workflow's typed handoff is the only path data travels.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named sequential-cv-tailor demonstrating the sequential-pipeline x hr-recruiting
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-hr-recruiting-sequential-cv-tailor. Java package
io.akka.samples.sequentialcvtailor. Akka 3.6.0. HTTP port 9856.

Components to wire (exactly):

- 2 AutonomousAgents:
  * CvGeneratorAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
    definition() returns AgentDefinition with .instructions(<system prompt loaded from
    prompts/cv-generator-agent.md>) and one .capability(TaskAcceptance.of(
    CvTasks.GENERATE_BASE_CV).maxIterationsPerTask(4)) entry. Function tools registered
    with .tools(ProfileTools.class). The before-call sanitizer (PiiSanitizer) is registered
    on this agent via the agent's sanitizer-configuration block.
  * CvTailorAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
    definition() returns AgentDefinition with .instructions(<system prompt loaded from
    prompts/cv-tailor-agent.md>) and one .capability(TaskAcceptance.of(
    CvTasks.TAILOR_CV).maxIterationsPerTask(4)) entry. Function tools registered with
    .tools(TailorTools.class). PiiSanitizer is also registered on this agent.

- 1 Workflow CvPipelineWorkflow per requestId with three steps:
  * generateStep — emits GenerationStarted on the entity, then calls componentClient
    .forAutonomousAgent(CvGeneratorAgent.class, "gen-" + requestId).runSingleTask(
      TaskDef.instructions("CandidateProfileId: " + candidateProfileId +
        "\nProfile: " + serialisedProfile + "\nTask: generate a structured BaseCv.")
        .metadata("requestId", requestId)
        .metadata("stage", "GENERATE")
        .taskType(CvTasks.GENERATE_BASE_CV)
    ). Reads result to get BaseCv. Writes CvRequestEntity.recordBaseCv(baseCv).
    WorkflowSettings.stepTimeout 60s.
  * tailorStep — emits TailoringStarted, then runSingleTask on CvTailorAgent with
    TaskDef.instructions(formatTailorContext(baseCv, jobPosting)) and metadata.stage =
    "TAILOR", taskType CvTasks.TAILOR_CV. Writes CvRequestEntity.recordTailoredCv(tailoredCv).
    stepTimeout 60s.
  * evalStep — runs the deterministic AlignmentScorer over (tailoredCv, jobPosting) and
    writes CvRequestEntity.recordAlignment(alignmentResult). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(CvPipelineWorkflow::error). The error step writes RequestFailed
  and ends.

- 1 EventSourcedEntity CvRequestEntity (one per requestId). State CvRequestRecord{requestId,
  candidateProfileId: Optional<String>, jobPostingId: Optional<String>,
  baseCv: Optional<BaseCv>, tailoredCv: Optional<TailoredCv>,
  alignmentResult: Optional<AlignmentResult>, sanitizationCount: int,
  status: CvRequestStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  CvRequestStatus enum: CREATED, GENERATING, GENERATED, TAILORING, TAILORED, SCORED, FAILED.
  Events: CvRequestCreated{candidateProfileId, jobPostingId}, GenerationStarted,
  BaseCvReady{baseCv}, TailoringStarted, TailoredCvReady{tailoredCv},
  QualityScored{alignmentResult}, SanitizationApplied{agent, removedCount, appliedAt},
  RequestFailed{reason}.
  Commands: create, startGeneration, recordBaseCv, startTailoring, recordTailoredCv,
  recordAlignment, recordSanitization, fail, getRequest. emptyState() returns
  CvRequestRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). sanitizationCount initialised to 0.

- 1 View CvRequestView with row type CvRequestRow that mirrors CvRequestRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes CvRequestEntity events.
  ONE query getAllRequests: SELECT * AS requests FROM cv_request_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * CvEndpoint at /api with POST /cv-requests (body {candidateProfileId, jobPostingId};
    mints requestId; calls CvRequestEntity.create(...); then starts CvPipelineWorkflow with
    id "cv-pipeline-" + requestId; returns {requestId}), GET /cv-requests (list from
    getAllRequests, sorted newest-first), GET /cv-requests/{id} (one row), GET
    /cv-requests/sse (Server-Sent Events from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- CvTasks.java declaring two Task<R> constants:
    GENERATE_BASE_CV = Task.name("Generate base CV").description("Expand and structure a
      CandidateProfile into a BaseCv").resultConformsTo(BaseCv.class);
    TAILOR_CV = Task.name("Tailor CV").description("Adapt a BaseCv for a specific
      JobPosting").resultConformsTo(TailoredCv.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- PiiSanitizer.java — implements the before-call sanitizer hook. Inspects the payload for
  the fields candidateName, email, phone, address, dateOfBirth (by field name, not shape).
  Replaces each present non-null field with "[REDACTED]". Counts replacements. If count > 0,
  calls CvRequestEntity.recordSanitization(requestId, count, agentClass.getSimpleName()).
  The requestId is read from the TaskDef metadata key "requestId". Never logs original
  values.

- ProfileTools.java — @FunctionTool expandSummary(CandidateProfile profile) -> String
  (expand rawSummary into a 3-4 sentence professional summary); @FunctionTool
  extractSkills(CandidateProfile profile) -> List<Skill> (parse rawSkills and infer level
  from years/experience clues in rawExperience); @FunctionTool formatExperience(
  CandidateProfile profile) -> List<ExperienceEntry> (structure each RawExperience into
  bullet-point ExperienceEntry). These read from src/main/resources/sample-data/candidates/
  *.json for deterministic offline output.

- TailorTools.java — @FunctionTool matchKeywords(BaseCv baseCv, JobPosting posting) ->
  List<KeywordMatch> (scan summary + experience bulletPoints for each required/preferred
  keyword; return one KeywordMatch per keyword); @FunctionTool rewriteSummary(BaseCv baseCv,
  List<KeywordMatch> matches, JobPosting posting) -> String (rewrite summary to front-load
  matched required keywords); @FunctionTool adjustExperience(BaseCv baseCv, JobPosting
  posting) -> List<ExperienceEntry> (reorder and lightly reword bullet points to surface
  required keywords).

- AlignmentScorer.java — pure deterministic logic (no LLM). Inputs: TailoredCv, JobPosting.
  Outputs: AlignmentResult with score, rationale, keywordsCovered, keywordsMissed. Four
  checks, one point per check satisfied, starting from base 1: (1) all requiredKeywords
  present in summary+experience text (case-insensitive); (2) >= 50% of preferredKeywords
  present; (3) experience.size() >= 1; (4) tailoredSummary non-empty and length > 50.
  Score range 1-5. Rationale names the largest gap (e.g., "Missing required keywords: Java,
  Spring Boot").

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9856 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/candidates/*.json — three candidate fixture files:
  alice-chen.json (Java engineer background), bob-ramirez.json (data science background),
  carla-hoffmann.json (product management background). Each carries a full CandidateProfile
  with all fields including PII fields (to demonstrate the sanitizer removing them).

- src/main/resources/sample-data/postings/*.json — three job posting fixture files:
  senior-java-engineer.json, senior-data-scientist.json, product-manager-platform.json.
  Each carries required and preferred keyword lists.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, E1) matching Section 8.
  Matching simplified_view list. regulation_anchors: [] for general baseline.

- risk-survey.yaml at the project root with sector = hr-recruiting, decisions.authority_level
  = recommend-only (recruiter reviews CV before submission), data.data_classes.pii = true
  (candidate names, contact details, DOB flow through the system), oversight.human_in_loop =
  true, operations.agent_count = 2, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "pii-leak-to-model", "keyword-hallucination",
  "missing-experience-entries", "alignment-score-mismatch"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/cv-generator-agent.md and prompts/cv-tailor-agent.md loaded as agent system
  prompts.

- README.md at project root. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of CV request cards with status pills; right = selected request detail
  with BaseCv section, TailoredCv section, alignment score chip, keyword-match chips, and
  sanitization-audit strip). Browser title exactly:
  <title>Akka Sample: Sequential CV Tailor</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        typed-correct outputs per Task (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return. Within a single task run the mock drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order.
- Per-task mock-response shapes for THIS blueprint:
    generate-base-cv.json — 3 BaseCv entries (one per candidate fixture), each with a
      3-4 sentence generatedSummary, 5-8 Skill entries, 2-4 ExperienceEntry items, and
      a tool_calls array containing expandSummary + extractSkills + formatExperience in
      order.
    tailor-cv.json — 3 TailoredCv entries paired with the 3 job posting fixtures. Each
      carries a tailoredSummary, skills, adjusted experience, and keywordMatches. Also
      one entry where all requiredKeywords are absent (for J3 validation).
- A MockModelProvider.seedFor(requestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: both AutonomousAgents extend akka.javasdk.agent.autonomous.AutonomousAgent.
  CvTasks.java MUST exist with both task constants.
- Lesson 4: generateStep stepTimeout 60s, tailorStep stepTimeout 60s, evalStep stepTimeout
  5s, error step stepTimeout 5s.
- Lesson 6: every nullable lifecycle field on CvRequestRecord is Optional<T>.
- Lesson 7: CvTasks.java with GENERATE_BASE_CV and TAILOR_CV is mandatory.
- Lesson 8: model names — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9856 declared in application.conf's akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata user-facing.
- Lesson 12: UI fits 1080px content column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and themeVariables.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never by NodeList index.
  Exactly five <section class="tab-panel"> elements.
- The two-agent invariant: there are exactly TWO AutonomousAgents (CvGeneratorAgent,
  CvTailorAgent). The on-decision eval is rule-based (AlignmentScorer.java) — no LLM call.
- Task dependency is carried by typed task results: generateStep writes BaseCv onto the
  entity, tailorStep reads it and builds the TAILOR task's instruction context from it.
  Neither agent holds multi-stage context.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
