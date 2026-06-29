# SPEC — basic-cv-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BasicCvAgent.
**One-line pitch:** A user submits a candidate profile; one AI agent reads the sanitized profile (passed as a task attachment, never as inline prompt text) and returns a structured `GeneratedCv` — a polished CV document with named sections, keyword list, and a structured-output variant ready for ATS ingestion.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the HR recruiting domain. One `CvGeneratorAgent` (AutonomousAgent) carries the entire generation; the surrounding components only prepare its input and record its output. One governance mechanism is wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw profile submission and the agent call — so the model never sees government identifiers, financial account numbers, undeclared medical details, or payment-card-like tokens that a candidate may have included in a free-text profile field.

The blueprint also shows the **structured-output variant**: a second generation mode where the agent returns a machine-readable `GeneratedCv` JSON object suitable for direct ATS import, not just formatted prose. Both modes use the same agent, the same task, and the same sanitizer — only the task instructions differ.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills a **Candidate Profile** form: full name, contact email (will be redacted), target role, work-history entries (title, company, start/end, bullets), education entries, skills list, and an optional free-text **additional notes** field (this is where PII most often hides).
2. The user selects an **Output mode**: `Prose CV` (formatted Markdown document) or `Structured CV` (JSON object with named sections).
3. The user clicks **Generate CV**. The UI POSTs to `/api/cv-requests` and receives a `requestId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SANITIZED` — the redacted profile is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `generateStep` completes. The card transitions to `GENERATING` then `CV_GENERATED`. The CV appears: for Prose mode, a formatted Markdown block; for Structured mode, a rendered JSON tree plus a copy-to-clipboard button.
6. The user can submit another profile; the live list keeps the history visible with the newest entry at the top.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CvEndpoint` | `HttpEndpoint` | `/api/cv-requests/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `CvEntity`, `CvView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CvEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → sanitized → generating → generated → failed. Source of truth. | `CvEndpoint`, `ProfileSanitizer`, `CvWorkflow` | `CvView` |
| `ProfileSanitizer` | `Consumer` | Subscribes to `ProfileSubmitted` events; redacts PII; calls `CvEntity.attachSanitized`. | `CvEntity` events | `CvEntity` |
| `CvWorkflow` | `Workflow` | One workflow per request. Steps: `awaitSanitizedStep` → `generateStep`. | started by `ProfileSanitizer` once sanitized event lands | `CvGeneratorAgent`, `CvEntity` |
| `CvGeneratorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives generation instructions (prose or structured) in the task definition and the sanitized profile as a task attachment; returns `GeneratedCv`. | invoked by `CvWorkflow` | returns cv |
| `CvView` | `View` | Read model: one row per request for the UI. | `CvEntity` events | `CvEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum OutputMode { PROSE, STRUCTURED }

record WorkHistoryEntry(
    String title,
    String company,
    String startDate,       // "YYYY-MM" format
    Optional<String> endDate,   // absent = present role
    List<String> bullets
) {}

record EducationEntry(
    String degree,
    String institution,
    String graduationYear
) {}

record CvRequest(
    String requestId,
    String candidateFullName,
    String contactEmail,         // redacted by sanitizer
    String targetRole,
    List<WorkHistoryEntry> workHistory,
    List<EducationEntry> education,
    List<String> skills,
    Optional<String> additionalNotes,   // free-text; PII often lands here
    OutputMode outputMode,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedProfile(
    String redactedAdditionalNotes,     // sanitized free-text field
    String redactedContactEmail,        // replaced with [REDACTED-EMAIL]
    List<String> piiCategoriesFound
) {}

record CvSection(
    String sectionTitle,
    String content           // Markdown prose or JSON-serialized sub-object
) {}

record GeneratedCv(
    String headline,         // one-line role title + value proposition
    List<CvSection> sections,
    List<String> keywords,
    OutputMode outputMode,
    Instant generatedAt
) {}

record CvRequestState(
    String requestId,
    Optional<CvRequest> request,
    Optional<SanitizedProfile> sanitized,
    Optional<GeneratedCv> generatedCv,
    CvStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CvStatus {
    SUBMITTED, SANITIZED, GENERATING, CV_GENERATED, FAILED
}
```

Events on `CvEntity`: `ProfileSubmitted`, `ProfileSanitized`, `GenerationStarted`, `CvGenerated`, `GenerationFailed`.

Every nullable lifecycle field on the `CvRequestState` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/cv-requests` — body `{ candidateFullName, contactEmail, targetRole, workHistory: [WorkHistoryEntry], education: [EducationEntry], skills: [String], additionalNotes?, outputMode, submittedBy }` → `{ requestId }`.
- `GET /api/cv-requests` — list all requests, newest-first.
- `GET /api/cv-requests/{id}` — one request.
- `GET /api/cv-requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: BasicCvAgent</title>`.

The App UI tab is a two-column layout: a left column with a candidate profile submission form and a live list of submitted requests (status pill + age); a right column with the selected request's detail — redacted profile preview, PII category chips, generated CV output (Markdown rendered for Prose mode; formatted JSON tree for Structured mode).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `ProfileSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, bank account numbers, and sensitive free-text markers (medical conditions, disability disclosures) from the raw profile before any LLM call. Records which categories were found. The raw profile is preserved on the entity for audit.

## 9. Agent prompts

- `CvGeneratorAgent` → `prompts/cv-generator.md`. The single decision-making LLM. System prompt instructs it to read the attached sanitized profile, understand the output mode (Prose or Structured), and return a `GeneratedCv` with a headline, named sections, and a keyword list.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seed profile in Prose mode; within 30 s the generated CV appears with a non-empty headline, at least three sections (Summary, Work History, Education), and a keywords list.
2. **J2** — User submits the same profile in Structured mode; the `GeneratedCv.sections` list contains the same sections but each `section.content` is valid JSON, not Markdown prose.
3. **J3** — A profile whose `additionalNotes` contains `NIN AB123456C` and `jane.doe@example.com` is submitted; the LLM call log shows only `[REDACTED-NINO]` and `[REDACTED-EMAIL]`; the entity's `request.contactEmail` retains the raw value for audit.
4. **J4** — A profile with an empty `workHistory` list produces a `GeneratedCv` whose Work History section contains a placeholder string; the generated CV status reaches `CV_GENERATED` without error.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named basic-cv-agent demonstrating the single-agent × hr-recruiting cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-hr-recruiting-basic-cv-agent. Java package
io.akka.samples.basicagentcvgenerator. Akka 3.6.0. HTTP port 9871.

Components to wire (exactly):

- 1 AutonomousAgent CvGeneratorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/cv-generator.md>) and
  .capability(TaskAcceptance.of(CvTasks.GENERATE_CV).maxIterationsPerTask(3)). The task
  receives the output mode and any formatting guidance in its instruction text and the
  sanitized profile as a task ATTACHMENT (NOT as inline prompt text —
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  GeneratedCv{headline: String, sections: List<CvSection>, keywords: List<String>,
  outputMode: OutputMode, generatedAt: Instant}. No guardrail is registered on this agent
  (structured output validation is handled downstream by CvWorkflow inspecting the returned
  GeneratedCv fields before calling CvEntity.recordGenerated).

- 1 Workflow CvWorkflow per requestId with two steps:
  * awaitSanitizedStep — polls CvEntity.getRequest every 1s; on
    requestState.sanitized().isPresent() advances to generateStep.
    WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * generateStep — emits GenerationStarted, then calls componentClient.forAutonomousAgent(
    CvGeneratorAgent.class, "cv-gen-" + requestId).runSingleTask(
      TaskDef.instructions(formatInstructions(requestState.request, requestState.request.outputMode()))
        .attachment("profile.json", serializeProfile(requestState).getBytes())
    ) — returns a taskId, then forTask(taskId).result(CvTasks.GENERATE_CV) to fetch the cv.
    Validates that headline and sections are non-empty before calling
    CvEntity.recordGenerated(generatedCv). On validation failure or LLM error calls
    CvEntity.fail(reason). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(CvWorkflow::error).
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity CvEntity (one per requestId). State CvRequestState{requestId: String,
  request: Optional<CvRequest>, sanitized: Optional<SanitizedProfile>,
  generatedCv: Optional<GeneratedCv>, status: CvStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. CvStatus enum: SUBMITTED, SANITIZED, GENERATING,
  CV_GENERATED, FAILED. Events: ProfileSubmitted{request}, ProfileSanitized{sanitized},
  GenerationStarted{}, CvGenerated{generatedCv}, GenerationFailed{reason}. Commands:
  submit, attachSanitized, markGenerating, recordGenerated, fail, getRequest.
  emptyState() returns CvRequestState.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer ProfileSanitizer subscribed to CvEntity events; on ProfileSubmitted runs a
  regex+heuristic redaction pipeline (emails, phone numbers, government IDs including
  national insurance numbers and social security numbers, payment-card-like tokens, bank
  account numbers, medical condition keywords via a small deny-list) over
  request.additionalNotes and request.contactEmail, computes the list of categories found,
  builds SanitizedProfile, then calls CvEntity.attachSanitized(sanitized). After
  attachSanitized lands, the same Consumer starts a CvWorkflow with id = "cv-" + requestId.

- 1 View CvView with row type CvRow (mirrors CvRequestState minus request.contactEmail and
  request.additionalNotes in their raw form — the view holds the sanitized forms for the UI).
  Table updater consumes CvEntity events. ONE query getAllRequests:
  SELECT * AS requests FROM cv_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * CvEndpoint at /api with POST /cv-requests (body {candidateFullName, contactEmail,
    targetRole, workHistory, education, skills, additionalNotes?, outputMode, submittedBy};
    mints requestId; calls CvEntity.submit; returns {requestId}), GET /cv-requests
    (list from getAllRequests, sorted newest-first), GET /cv-requests/{id} (one row), GET
    /cv-requests/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- CvTasks.java declaring one Task<R> constant: GENERATE_CV = Task.name("Generate CV")
  .description("Read the attached sanitized candidate profile and produce a GeneratedCv
  with headline, named sections, and a keywords list")
  .resultConformsTo(GeneratedCv.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records OutputMode, WorkHistoryEntry, EducationEntry, CvRequest, SanitizedProfile,
  CvSection, GeneratedCv, CvRequestState, CvStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9871 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CvGeneratorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-profiles.jsonl with 3 seeded candidate profiles:
  a software engineer (5 years experience, Python/Java stack), a marketing manager (8 years,
  B2B SaaS), and a recent graduate (1 year internship, Data Science). Each contains 2–3
  PII strings in the additionalNotes field so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (S1) matching the mechanism in Section 8
  of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = recommend-only
  (the generated CV is a draft; a recruiter reviews before sending), oversight.human_in_loop
  = true (recruiter reads every generated CV before forwarding), failure.failure_modes
  including "hallucinated-experience", "missing-section", "pii-leakage-via-llm",
  "tone-mismatch"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/cv-generator.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: BasicCvAgent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of request cards; right = selected-request detail with
  sanitized profile preview, PII category chips, generated CV output).
  Browser title exactly: <title>Akka Sample: BasicCvAgent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-cv.json — 6 GeneratedCv entries: 4 valid (two Prose mode, two Structured mode)
    covering different candidate archetypes; 1 Prose entry with an empty sections list
    (exercises the downstream validation in generateStep and transitions the entity to
    FAILED); 1 Structured entry with headline present but all section.content strings
    set to empty (also triggers the failure path). The mock should select the invalid
    entries deterministically when (seedFor(requestId) % 5 == 0) so J4 is reproducible.
- A MockModelProvider.seedFor(requestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CvGeneratorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CvTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (generateStep
  60s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the CvRequestState record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: CvTasks.java with GENERATE_CV = Task.name(...).description(...)
  .resultConformsTo(GeneratedCv.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9871 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (CvGeneratorAgent).
  There is no second LLM call anywhere in the system.
- The profile is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated generateStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
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
