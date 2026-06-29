# SPEC — haiku-subagents-under-opus

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Haiku Sub-Agents under Opus (multimodal).
**One-line pitch:** Submit a batch of images with a question; an Opus coordinator delegates per-image analysis to fast, cheap Haiku sub-agents in parallel, then synthesises their reports into one answer.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern for cost-optimised multimodal workloads: an Opus coordinator owns reasoning, planning, and synthesis; lightweight Haiku sub-agents handle the repetitive per-image analysis in parallel. The blueprint also demonstrates a **PII sanitizer** that strips faces and document text from base64 image payloads before they reach sub-agents, and an **output guardrail** that vets the coordinator's synthesised answer before it is returned to the caller.

## 3. User-facing flows

The user opens the App UI tab, enters a question, attaches 1–8 images (or lets the simulator inject a canned batch), and submits.

1. The system creates an `AnalysisJob` record in `QUEUED` and starts an `AnalysisWorkflow`.
2. A `PiiSanitizer` redacts any PII from each image payload (faces blurred, document text blanked) before the workflow fans out.
3. The `OpusCoordinator` decomposes the question into a per-image instruction: `ImageTask { instruction, imageRef }`.
4. The workflow fans out: one `HaikuImageAnalyst` per image, all running concurrently.
5. Each `HaikuImageAnalyst` returns an `ImageReport { imageRef, description, tags, detectedLabels, analysedAt }`.
6. The coordinator synthesises the per-image reports into a `SynthesisedAnswer { answer, imageSummaries, guardrailVerdict, synthesisedAt }`.
7. An output guardrail vets the synthesised answer; if it fails, the job moves to `BLOCKED`. Otherwise, the job moves to `SYNTHESISED`.
8. If any sub-agent fails or times out after 45 seconds, the workflow short-circuits: the coordinator synthesises from whichever reports arrived, and the job enters `PARTIAL`.

A `BatchSimulator` (TimedAction) drips a canned 3-image batch every 90 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OpusCoordinator` | `AutonomousAgent` | Decomposes the image batch question into per-image tasks; synthesises sub-agent reports into the final answer; runs the output guardrail. | `AnalysisWorkflow` | returns typed result to workflow |
| `HaikuImageAnalyst` | `AutonomousAgent` | Analyses a single image and returns an `ImageReport`. Uses the Haiku model for cost efficiency. | `AnalysisWorkflow` | — |
| `PiiSanitizer` | component (not an agent) | Redacts PII from base64 image payloads before sub-agent calls. Called synchronously within `AnalysisWorkflow`. | `AnalysisWorkflow` | `AnalysisWorkflow` |
| `AnalysisWorkflow` | `Workflow` | Orchestrates PII sanitization, per-image fan-out, report join, synthesis, and guardrail. | `AnalysisEndpoint`, `BatchRequestConsumer` | `AnalysisJobEntity` |
| `AnalysisJobEntity` | `EventSourcedEntity` | Holds the job lifecycle (QUEUED → ANALYSING → SYNTHESISED / PARTIAL / BLOCKED). | `AnalysisWorkflow` | `AnalysisView` |
| `BatchQueue` | `EventSourcedEntity` | Logs each submitted batch for replay/audit. | `AnalysisEndpoint`, `BatchSimulator` | `BatchRequestConsumer` |
| `AnalysisView` | `View` | List-of-jobs read model. | `AnalysisJobEntity` events | `AnalysisEndpoint` |
| `BatchRequestConsumer` | `Consumer` | Listens to `BatchQueue` events and starts a workflow per submission. | `BatchQueue` events | `AnalysisWorkflow` |
| `BatchSimulator` | `TimedAction` | Drips a canned 3-image batch every 90 s. | scheduler | `BatchQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised job every 5 minutes for eval scoring; emits a `JobEvalScored` event. | scheduler | `AnalysisJobEntity` |
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/analysis/*` — submit, get, list, SSE. | — | `AnalysisView`, `BatchQueue`, `AnalysisJobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record BatchSubmission(List<ImagePayload> images, String question, String submittedBy) {}
record ImagePayload(String imageRef, String base64Data, String mimeType) {}
record SanitizedImage(String imageRef, String sanitizedBase64Data, String mimeType,
                      boolean piiDetected) {}

record ImageTask(String imageRef, String instruction) {}

record DetectedLabel(String label, double confidence) {}
record ImageReport(String imageRef, String description, List<String> tags,
                   List<DetectedLabel> detectedLabels, Instant analysedAt) {}

record SynthesisedAnswer(String answer, List<ImageReport> imageSummaries,
                         String guardrailVerdict, Instant synthesisedAt) {}

record AnalysisJob(
    String jobId,
    String question,
    List<String> imageRefs,
    JobStatus status,
    Optional<List<SanitizedImage>> sanitizedImages,
    Optional<List<ImageReport>> reports,
    Optional<SynthesisedAnswer> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { QUEUED, ANALYSING, SYNTHESISED, PARTIAL, BLOCKED }
```

### Events (on `AnalysisJobEntity`)

`JobCreated`, `ImagesSubmitted`, `ImagesSanitized`, `AnalysisStarted`, `ReportAttached`, `JobSynthesised`, `JobPartial`, `JobBlocked`, `JobEvalScored`.

### Events (on `BatchQueue`)

`BatchSubmitted { jobId, imageCount, question, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/analysis` — body `{ question, images: [{ imageRef, base64Data, mimeType }] }` → `{ jobId }`. Starts a workflow.
- `GET /api/analysis` — list all jobs. Optional `?status=QUEUED|ANALYSING|SYNTHESISED|PARTIAL|BLOCKED`.
- `GET /api/analysis/{id}` — one job.
- `GET /api/analysis/sse` — server-sent events stream of every job change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Haiku Sub-Agents under Opus (multimodal)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a question + images (file input or URL list), live list of jobs with status pills, expand-row to see per-image reports + synthesised answer + eval score.

Browser title: `<title>Akka Sample: Haiku Sub-Agents under Opus (multimodal)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii` flavor, runs before sub-agent calls): `PiiSanitizer` strips faces and document text from base64 image payloads before any `HaikuImageAnalyst` call. Non-blocking in the sense that analysis still proceeds with the sanitized images — the workflow does not abort when PII is detected; it redacts and continues.
- **G1 — output guardrail** (`before-agent-response` on `OpusCoordinator`): vets the synthesised answer for references to content that should have been redacted and for factual inconsistencies. Blocking. Failure → `BLOCKED`.

## 9. Agent prompts

- `OpusCoordinator` → `prompts/opus-coordinator.md`. Decomposes the question into per-image tasks; later synthesises all image reports into one answer.
- `HaikuImageAnalyst` → `prompts/haiku-image-analyst.md`. Analyses a single image; returns `ImageReport`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a 3-image batch; job progresses QUEUED → ANALYSING → SYNTHESISED within 90 s; UI reflects each transition via SSE; all three `ImageReport` rows appear in the expanded row.
2. **J2** — Inject a sub-agent timeout (set `HaikuImageAnalyst` step timeout to 1 s); job enters PARTIAL with whichever reports arrived.
3. **J3** — Submit an image with a face; confirm the sanitizer ran (`piiDetected = true`) and the synthesised answer contains no description of the face.
4. **J4** — Inject a guardrail failure (coordinator output references redacted content); job enters BLOCKED.
5. **J5** — Wait for `EvalSampler`; the job shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named haiku-subagents-under-opus demonstrating the
delegation-supervisor-workers × research-intel cell. Requires Anthropic API
key (Opus + Haiku models). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-research-intel-opus-haiku-subagents.
Java package io.akka.samples.haikusubagentsunderopusmultimodal. Akka 3.6.0.
HTTP port 9850.

Components to wire (exactly):
- 2 AutonomousAgents:
  * OpusCoordinator — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/opus-coordinator.md.
    Returns List<ImageTask>{imageRef, instruction} for DECOMPOSE
    and SynthesisedAnswer{answer, imageSummaries, guardrailVerdict, synthesisedAt}
    for SYNTHESISE.
    Model: claude-opus-4-5 (or latest Opus available via ${?ANTHROPIC_API_KEY}).

  * HaikuImageAnalyst — capability(TaskAcceptance.of(ANALYSE_IMAGE)
    .maxIterationsPerTask(2)). System prompt from prompts/haiku-image-analyst.md.
    Returns ImageReport{imageRef, description, tags: List<String>,
    detectedLabels: List<DetectedLabel{label, confidence}>, analysedAt}.
    Model: claude-haiku-4-5 (or latest Haiku available via ${?ANTHROPIC_API_KEY}).

- 1 plain component PiiSanitizer (not an AutonomousAgent) with method
  SanitizedImage sanitize(ImagePayload). Marks piiDetected=true and blanks
  base64Data when heuristic detects a face (data URL width/height ratio
  heuristic) or embedded document text (base64 contains common document
  magic bytes). In the real system this is a placeholder that always returns
  piiDetected=false with the original data — the hook is there for a real
  redaction library to be dropped in.

- 1 Workflow AnalysisWorkflow with steps:
  sanitizeStep -> decomposeStep -> [parallel] analyseStep(per image) ->
  joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  sanitizeStep calls PiiSanitizer.sanitize for each image (sequential, not
  parallel — order must be preserved for imageRef matching).
  decomposeStep calls forAutonomousAgent(OpusCoordinator.class, DECOMPOSE)
  passing the sanitized images and question.
  Each analyseStep calls forAutonomousAgent(HaikuImageAnalyst.class,
  ANALYSE_IMAGE) with one SanitizedImage + the matching ImageTask.instruction;
  all run in parallel via CompletionStage.allOf.
  WorkflowSettings.builder().stepTimeout(AnalysisWorkflow::analyseStep,
  ofSeconds(45)) for each image step. On any step timeout or agent failure,
  route to partialStep that calls synthesiseStep with whichever ImageReports
  arrived and ends with JobPartial. synthesiseStep calls
  forAutonomousAgent(OpusCoordinator.class, SYNTHESISE) with the full
  report list; give synthesiseStep a 90s stepTimeout. guardrailStep runs a
  deterministic vetter checking that no SanitizedImage with piiDetected=true
  is referenced by description text in the SynthesisedAnswer; on failure,
  end with JobBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity AnalysisJobEntity holding state AnalysisJob{jobId,
  question, imageRefs: List<String>, JobStatus, Optional<List<SanitizedImage>>
  sanitizedImages, Optional<List<ImageReport>> reports,
  Optional<SynthesisedAnswer> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. JobStatus enum: QUEUED, ANALYSING, SYNTHESISED,
  PARTIAL, BLOCKED. Events: JobCreated, ImagesSubmitted, ImagesSanitized,
  AnalysisStarted, ReportAttached, JobSynthesised, JobPartial, JobBlocked,
  JobEvalScored. Commands: createJob, submitImages, markSanitized, startAnalysis,
  attachReport, synthesise, markPartial, block, recordEval, getJob.
  emptyState() returns AnalysisJob.initial("", null) with no commandContext()
  reference.

- 1 EventSourcedEntity BatchQueue with command enqueueBatch(images, question,
  submittedBy) emitting BatchSubmitted{jobId, imageCount, question,
  submittedBy, submittedAt}.

- 1 View AnalysisView with row type AnalysisJobRow (mirrors AnalysisJob minus
  heavy nested payloads; every nullable field is Optional<T>). Table updater
  consumes AnalysisJobEntity events. ONE query getAllJobs SELECT * AS jobs
  FROM analysis_view. No WHERE status filter — caller filters client-side.

- 1 Consumer BatchRequestConsumer subscribed to BatchQueue events; on
  BatchSubmitted starts an AnalysisWorkflow with the jobId as the workflow id.

- 2 TimedActions:
  * BatchSimulator — every 90s, reads next batch from
    src/main/resources/sample-events/image-batches.jsonl and calls
    BatchQueue.enqueueBatch.
  * EvalSampler — every 5 minutes, queries AnalysisView.getAllJobs, picks
    the oldest SYNTHESISED job without an evalScore, runs a 1–5 rubric
    judge over the synthesised answer, then calls
    AnalysisJobEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /analysis, GET /analysis,
    GET /analysis/{id}, GET /analysis/sse, and the /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- AnalysisTasks.java declaring three Task<R> constants: DECOMPOSE
  (List<ImageTask>), ANALYSE_IMAGE (ImageReport), SYNTHESISE (SynthesisedAnswer).
- Domain records BatchSubmission, ImagePayload, SanitizedImage, ImageTask,
  DetectedLabel, ImageReport, SynthesisedAnswer.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9850
  and akka.javasdk.agent model-provider blocks for anthropic with two model
  entries: coordinator model = claude-opus-4-5 and analyst model = claude-haiku-4-5,
  both api-key read from ${?ANTHROPIC_API_KEY}. Also include openai (gpt-4o) and
  googleai-gemini (gemini-2.5-flash) blocks as fallback options.
- src/main/resources/sample-events/image-batches.jsonl with 6 canned batch lines
  (each is a JSON object with question and a list of 2–4 imageRefs pointing to
  placeholder data URLs in src/main/resources/sample-images/).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 pii sanitizer,
  G1 output guardrail before-agent-response) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  multimodal-image-analysis, decisions.authority_level = recommend-only,
  data.data_classes.pii = true (images may contain PII),
  capabilities.synthetic_media = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/opus-coordinator.md, prompts/haiku-image-analyst.md loaded at agent
  startup as system prompts.
- README.md at the project root: title
  "Akka Sample: Haiku Sub-Agents under Opus (multimodal)", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid
  diagrams + click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval
  Matrix (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows), App UI (form with question input + image file/URL list,
  live list of jobs with status pills, expand-row to see per-image reports +
  synthesised answer + eval score). Browser title exactly:
  <title>Akka Sample: Haiku Sub-Agents under Opus (multimodal)</title>. No
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
  the ModelProvider interface with a per-agent dispatch on the agent class name.
- Per-agent mock-response shapes for THIS blueprint:
    opus-coordinator.json — list of either List<ImageTask> or SynthesisedAnswer
      objects. 4–6 decompose entries (each a list of 2–4 ImageTask objects with
      instruction strings) and 4–6 SynthesisedAnswer entries (each with an
      80–150 word answer, 2–4 imageSummary stubs, guardrailVerdict = "ok").
    haiku-image-analyst.json — 6–10 ImageReport entries, each with a
      3-sentence description, 3–5 string tags (e.g., "outdoor", "vehicle",
      "daytime"), and 2–4 DetectedLabel entries with confidence 0.7–0.99.
- A MockModelProvider.seedFor(jobId) helper makes the selection deterministic
  per job id so the same job produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout
  (45s per image analyst, 90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion AnalysisTasks.java declaring
  Task<R> constants.
- Lesson 8: verify model names current before locking (claude-opus-4-5,
  claude-haiku-4-5, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9850 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Requires Anthropic API key for
  Opus + Haiku models"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  theme variables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state names
  render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  NodeList index; delete removed panels from the DOM, do not display:none them.
- Parallel fan-out for per-image steps uses CompletionStage.allOf, NOT sequential
  calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, marketing tone.
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
