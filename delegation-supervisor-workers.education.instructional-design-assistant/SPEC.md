# SPEC — instructional-design-assistant

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Instructional Design Assistant.
**One-line pitch:** Describe a learning unit; a supervisor delegates outline, materials, and assessment work to specialists in parallel, assembles the package, and gates publication on faculty approval.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their outputs, and asks a fourth AutonomousAgent to assemble a publishable unit package. The blueprint also demonstrates a **human-in-the-loop** application gate (faculty review before the unit is published) and a **before-agent-response guardrail** that checks assembled materials for academic-integrity violations.

## 3. User-facing flows

The user opens the App UI tab and submits a unit-design request (subject, grade level, learning objectives) via the form.

1. The system creates a `UnitPackage` record in `DRAFT` and starts a `UnitDesignWorkflow`.
2. The Supervisor decomposes the request into three parallel work items: an outline task, a materials task, and an assessment task.
3. The workflow forks: all three specialists run concurrently. Each returns a typed payload.
4. The Supervisor merges the three payloads into an `AssembledUnit { outline, materials, assessment, integrityVerdict }`.
5. A before-agent-response guardrail vets the assembled content for academic-integrity violations; if it fails, the unit moves to `BLOCKED`. Otherwise, the unit moves to `PENDING_REVIEW`.
6. A faculty reviewer reads the unit in the App UI and either approves (→ `PUBLISHED`) or rejects with feedback (→ `REVISION_REQUESTED`). A rejected unit re-enters the design cycle from the beginning.
7. If any specialist times out after 60 seconds, the workflow short-circuits: the Supervisor assembles from whichever sides returned, and the unit enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample unit-design request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `UnitDesignSupervisor` | `AutonomousAgent` | Decomposes the request into three work items; later merges specialist outputs into an assembled unit package. | `UnitDesignWorkflow` | returns typed result to workflow |
| `OutlineSpecialist` | `AutonomousAgent` | Drafts the learning-objective outline and topic sequence. | `UnitDesignWorkflow` | — |
| `MaterialsSpecialist` | `AutonomousAgent` | Authors lesson content, activities, and resources. | `UnitDesignWorkflow` | — |
| `AssessmentSpecialist` | `AutonomousAgent` | Writes quiz questions, rubrics, and grading criteria. | `UnitDesignWorkflow` | — |
| `UnitDesignWorkflow` | `Workflow` | Coordinates parallel fan-out, assembly, guardrail, and the faculty review gate. | `UnitDesignEndpoint`, `DesignRequestConsumer` | `UnitPackageEntity`, `FacultyReviewQueue` |
| `UnitPackageEntity` | `EventSourcedEntity` | Holds the unit's lifecycle (draft → in-progress → pending-review → published / revision-requested / degraded / blocked). | `UnitDesignWorkflow` | `UnitPackageView` |
| `FacultyReviewQueue` | `EventSourcedEntity` | Tracks pending faculty decisions; records approvals and rejections. | `UnitDesignWorkflow`, `UnitDesignEndpoint` | `DesignRequestConsumer` |
| `UnitPackageView` | `View` | List-of-units read model. | `UnitPackageEntity` events | `UnitDesignEndpoint` |
| `DesignRequestConsumer` | `Consumer` | Listens to `DesignRequestQueue` events and starts a workflow per submission. Also listens to `FacultyReviewQueue` events for rejection re-cycles. | `DesignRequestQueue` events, `FacultyReviewQueue` events | `UnitDesignWorkflow` |
| `DesignRequestQueue` | `EventSourcedEntity` | Logs each submitted design request for replay/audit. | `UnitDesignEndpoint`, `RequestSimulator` | `DesignRequestConsumer` |
| `RequestSimulator` | `TimedAction` | Drips a sample unit-design request every 60 s. | scheduler | `DesignRequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one published unit every 5 minutes for pedagogical quality scoring; emits a `QualityScored` event. | scheduler | `UnitPackageEntity` |
| `UnitDesignEndpoint` | `HttpEndpoint` | `/api/units/*` — submit, get, list, SSE, faculty review. | — | `UnitPackageView`, `DesignRequestQueue`, `UnitPackageEntity`, `FacultyReviewQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record UnitDesignRequest(String subject, String gradeLevel, List<String> learningObjectives,
                         String requestedBy) {}

record OutlineDraft(List<OutlineSection> sections, Instant draftedAt) {}
record OutlineSection(String title, String purpose, List<String> subtopics) {}

record MaterialsBundle(List<LessonActivity> activities, List<String> resourceLinks,
                       Instant authoredAt) {}
record LessonActivity(String activityTitle, String description, String activityType) {}

record AssessmentPack(List<QuizQuestion> questions, String rubricSummary, Instant assessedAt) {}
record QuizQuestion(String prompt, List<String> options, String correctAnswer,
                    String bloomsLevel) {}

record DesignWorkPlan(String outlineTask, String materialsTask, String assessmentTask) {}

record AssembledUnit(OutlineDraft outline, MaterialsBundle materials,
                     AssessmentPack assessment, String integrityVerdict,
                     Instant assembledAt) {}

record UnitPackage(
    String unitId,
    String subject,
    String gradeLevel,
    UnitStatus status,
    Optional<OutlineDraft> outline,
    Optional<MaterialsBundle> materials,
    Optional<AssessmentPack> assessment,
    Optional<AssembledUnit> assembled,
    Optional<String> facultyFeedback,
    Optional<String> failureReason,
    Optional<Integer> qualityScore,
    Optional<String> qualityRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum UnitStatus { DRAFT, IN_PROGRESS, PENDING_REVIEW, PUBLISHED,
                  REVISION_REQUESTED, DEGRADED, BLOCKED }
```

### Events (on `UnitPackageEntity`)

`UnitCreated`, `OutlineAttached`, `MaterialsAttached`, `AssessmentAttached`, `UnitAssembled`, `UnitPendingReview`, `UnitPublished`, `RevisionRequested`, `UnitDegraded`, `UnitBlocked`, `QualityScored`.

### Events (on `DesignRequestQueue`)

`DesignRequestSubmitted { unitId, subject, gradeLevel, learningObjectives, requestedBy, submittedAt }`.

### Events (on `FacultyReviewQueue`)

`ReviewDecisionRecorded { unitId, decision, feedback, reviewedBy, decidedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/units` — body `{ subject, gradeLevel, learningObjectives }` → `{ unitId }`. Starts a workflow.
- `GET /api/units` — list all units. Optional `?status=DRAFT|IN_PROGRESS|PENDING_REVIEW|PUBLISHED|...`.
- `GET /api/units/{id}` — one unit.
- `GET /api/units/sse` — server-sent events stream of every unit change.
- `POST /api/units/{id}/review` — body `{ decision: "approve"|"reject", feedback?, reviewedBy }` → unit moves to PUBLISHED or REVISION_REQUESTED.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Instructional Design Assistant"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a unit-design request, live list of units with status pills, faculty review controls for PENDING_REVIEW units, expand-row to see outline, materials, assessment, and quality score.

Browser title: `<title>Akka Sample: Instructional Design Assistant</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — faculty review gate** (`hitl · application`): after assembly and guardrail pass, the workflow parks at a `pendingReviewStep` that suspends until `FacultyReviewQueue` records a decision. A faculty approval moves the unit to `PUBLISHED`; a rejection moves it to `REVISION_REQUESTED` and re-queues the design request. Blocking.
- **G1 — academic-integrity guardrail** (`guardrail · before-agent-response` on `UnitDesignSupervisor`): vets the assembled unit content for plagiarism indicators, copyright issues, and prohibited content before faculty review begins. Failure → `BLOCKED`.

## 9. Agent prompts

- `UnitDesignSupervisor` → `prompts/unit-design-supervisor.md`. Decomposes the request; later assembles outputs into the final unit package.
- `OutlineSpecialist` → `prompts/outline-specialist.md`. Drafts the learning-objective outline; returns `OutlineDraft`.
- `MaterialsSpecialist` → `prompts/materials-specialist.md`. Authors lesson content; returns `MaterialsBundle`.
- `AssessmentSpecialist` → `prompts/assessment-specialist.md`. Writes assessment items; returns `AssessmentPack`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a unit-design request; unit progresses DRAFT → IN_PROGRESS → PENDING_REVIEW → PUBLISHED within 90 s (after simulated faculty approval); UI reflects each transition via SSE.
2. **J2** — Inject a specialist timeout (set `MaterialsSpecialist` timeout to 1 s); unit enters DEGRADED.
3. **J3** — Inject a guardrail failure; unit enters BLOCKED before reaching PENDING_REVIEW.
4. **J4** — Faculty rejects the unit with feedback; unit enters REVISION_REQUESTED; a new cycle begins.
5. **J5** — Wait after a published unit; the unit's row shows a quality score from EvalSampler.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named instructional-design-assistant demonstrating the
delegation-supervisor-workers × education cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-education-instructional-design-assistant.
Java package io.akka.samples.instructionaldesignassistant. Akka 3.6.0.
HTTP port 9710.

Components to wire (exactly):
- 4 AutonomousAgents:
  * UnitDesignSupervisor — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(ASSEMBLE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/unit-design-supervisor.md. Returns DesignWorkPlan{outlineTask, materialsTask,
    assessmentTask} for DECOMPOSE and AssembledUnit{outline, materials, assessment,
    integrityVerdict, assembledAt} for ASSEMBLE.
  * OutlineSpecialist — capability(TaskAcceptance.of(DRAFT_OUTLINE).maxIterationsPerTask(3)).
    System prompt from prompts/outline-specialist.md. Returns
    OutlineDraft{sections: List<OutlineSection{title, purpose, subtopics}>, draftedAt}.
  * MaterialsSpecialist — capability(TaskAcceptance.of(CREATE_MATERIALS).maxIterationsPerTask(3)).
    System prompt from prompts/materials-specialist.md. Returns
    MaterialsBundle{activities: List<LessonActivity{activityTitle, description, activityType}>,
    resourceLinks: List<String>, authoredAt}.
  * AssessmentSpecialist — capability(TaskAcceptance.of(BUILD_ASSESSMENT).maxIterationsPerTask(3)).
    System prompt from prompts/assessment-specialist.md. Returns
    AssessmentPack{questions: List<QuizQuestion{prompt, options, correctAnswer, bloomsLevel}>,
    rubricSummary: String, assessedAt}.

- 1 Workflow UnitDesignWorkflow with steps:
  decomposeStep -> [parallel] outlineStep, materialsStep, assessmentStep
  -> joinStep -> assembleStep -> guardrailStep -> pendingReviewStep -> publishStep|reviseStep.
  decomposeStep calls forAutonomousAgent(UnitDesignSupervisor.class, DECOMPOSE).
  outlineStep, materialsStep, and assessmentStep run in parallel (CompletionStage allOf);
  each governed by WorkflowSettings.builder()
    .stepTimeout(UnitDesignWorkflow::outlineStep, ofSeconds(60))
    .stepTimeout(UnitDesignWorkflow::materialsStep, ofSeconds(60))
    .stepTimeout(UnitDesignWorkflow::assessmentStep, ofSeconds(60)).
  On any timeout, transition to degradeStep which calls assembleStep with whichever
  sides returned, ends with UnitDegraded.
  assembleStep calls forAutonomousAgent(UnitDesignSupervisor.class, ASSEMBLE) with the
  three merged inputs; give assembleStep a 90s stepTimeout.
  guardrailStep runs the deterministic vetter + LLM judge on the assembled content;
  on failure, end with UnitBlocked.
  pendingReviewStep is a durable pause: write UnitPendingReview to UnitPackageEntity,
  then pause the workflow until FacultyReviewQueue emits a ReviewDecisionRecorded event
  for this unitId. On approval, call UnitPackageEntity.publish (PUBLISHED). On rejection,
  call UnitPackageEntity.requestRevision (REVISION_REQUESTED) and re-enqueue to
  DesignRequestQueue with the faculty feedback appended to the objectives. WorkflowSettings
  is nested inside Workflow — no import.

- 1 EventSourcedEntity UnitPackageEntity holding state UnitPackage{unitId, subject,
  gradeLevel, UnitStatus, Optional<OutlineDraft> outline, Optional<MaterialsBundle> materials,
  Optional<AssessmentPack> assessment, Optional<AssembledUnit> assembled,
  Optional<String> facultyFeedback, Optional<String> failureReason,
  Optional<Integer> qualityScore, Optional<String> qualityRationale, Instant createdAt,
  Optional<Instant> finishedAt}. UnitStatus enum: DRAFT, IN_PROGRESS, PENDING_REVIEW,
  PUBLISHED, REVISION_REQUESTED, DEGRADED, BLOCKED. Events: UnitCreated, OutlineAttached,
  MaterialsAttached, AssessmentAttached, UnitAssembled, UnitPendingReview, UnitPublished,
  RevisionRequested, UnitDegraded, UnitBlocked, QualityScored. Commands: createUnit,
  attachOutline, attachMaterials, attachAssessment, assemble, pendingReview, publish,
  requestRevision, degrade, block, recordQuality, getUnit. emptyState() returns
  UnitPackage.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity DesignRequestQueue with command enqueueRequest(subject, gradeLevel,
  learningObjectives, requestedBy) emitting DesignRequestSubmitted{unitId, subject,
  gradeLevel, learningObjectives, requestedBy, submittedAt}.

- 1 EventSourcedEntity FacultyReviewQueue with command recordDecision(unitId, decision,
  feedback, reviewedBy) emitting ReviewDecisionRecorded{unitId, decision, feedback,
  reviewedBy, decidedAt}.

- 1 View UnitPackageView with row type UnitPackageRow (mirrors UnitPackage minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  UnitPackageEntity events. ONE query getAllUnits SELECT * AS units FROM unit_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer DesignRequestConsumer subscribed to DesignRequestQueue events; on
  DesignRequestSubmitted starts a UnitDesignWorkflow with the unitId as the workflow id.
  Also subscribed to FacultyReviewQueue events; on ReviewDecisionRecorded with
  decision=reject re-enqueues the unit design request with faculty feedback.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/unit-design-requests.jsonl and calls
    DesignRequestQueue.enqueueRequest.
  * EvalSampler — every 5 minutes, queries UnitPackageView.getAllUnits, picks the oldest
    PUBLISHED unit without a qualityScore, runs a 1–5 pedagogical rubric judge over the
    assembled content, then calls UnitPackageEntity.recordQuality(score, rationale).

- 2 HttpEndpoints:
  * UnitDesignEndpoint at /api with POST /units, GET /units, GET /units/{id},
    GET /units/sse, POST /units/{id}/review, and the /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- UnitDesignTasks.java declaring five Task<R> constants: DECOMPOSE (DesignWorkPlan),
  DRAFT_OUTLINE (OutlineDraft), CREATE_MATERIALS (MaterialsBundle),
  BUILD_ASSESSMENT (AssessmentPack), ASSEMBLE (AssembledUnit).
- Domain records DesignWorkPlan, OutlineSection, OutlineDraft, LessonActivity,
  MaterialsBundle, QuizQuestion, AssessmentPack, AssembledUnit, UnitDesignRequest.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9710 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/unit-design-requests.jsonl with 8 canned unit-design
  request lines covering subjects like biology, algebra, world history, and creative writing.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 hitl application,
  G1 guardrail before-agent-response) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = education,
  decisions.authority_level = recommend-only, data.pii = false,
  capabilities.content_generation = true; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/unit-design-supervisor.md, prompts/outline-specialist.md,
  prompts/materials-specialist.md, prompts/assessment-specialist.md loaded at agent
  startup as system prompts.
- README.md at the project root: title "Akka Sample: Instructional Design Assistant",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows),
  App UI (form + live list with status pills + faculty review controls for PENDING_REVIEW
  units). Browser title exactly: <title>Akka Sample: Instructional Design Assistant</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (unit-design-supervisor.json,
  outline-specialist.json, materials-specialist.json, assessment-specialist.json),
  picks one entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    unit-design-supervisor.json — list of either DesignWorkPlan or AssembledUnit objects.
      4–6 DesignWorkPlan entries (outlineTask + materialsTask + assessmentTask triples)
      and 4–6 AssembledUnit entries (each with full outline, materials, assessment, and
      integrityVerdict = "ok").
    outline-specialist.json — 4–6 OutlineDraft entries, each with 3–5 OutlineSection
      items covering realistic K-12 or university topic breakdowns.
    materials-specialist.json — 4–6 MaterialsBundle entries, each with 3–6
      LessonActivity items of varying activityType (lecture, lab, discussion, project).
    assessment-specialist.json — 4–6 AssessmentPack entries, each with 4–6 QuizQuestion
      items at different bloomsLevel values (remember, understand, apply, analyse).
- A MockModelProvider.seedFor(unitId) helper makes the selection deterministic per unit
  id so the same unit produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s
  specialists, 90s assembly); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion UnitDesignTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9710 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList
  index; delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK
  in narrative, T1/T2/T3/T4, deferred, marketing tone.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
