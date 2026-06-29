# SPEC — akka-parallel-best-of-n

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Parallel Best-of-N.
**One-line pitch:** Submit a source text; a supervisor fans the job out to N parallel translation workers, collects all variants, and selects the best one.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three `TranslationWorker` AutonomousAgent branches concurrently, collects their variant outputs, and asks a `TranslationSupervisor` AutonomousAgent to select the best. The blueprint also demonstrates an **output guardrail** that vets the selected translation before it is returned to the caller, and an **eval-event** that logs the supervisor's selection decision for quality tracking.

## 3. User-facing flows

The user opens the App UI tab and submits a source text and target language via the form.

1. The system creates a `TranslationJob` record in `PENDING` and starts a `TranslationWorkflow`.
2. The Supervisor decomposes the job into three variant instructions: formal register, informal register, and literal register.
3. The workflow forks: all three `TranslationWorker` branches run concurrently. Each returns a `TranslationVariant`.
4. The Supervisor receives all variants and selects the best one, producing a `SelectionResult { selectedVariant, selectionRationale, guardrailVerdict }`.
5. An output guardrail vets the selected translation; if it fails, the job moves to `BLOCKED`. Otherwise, the job moves to `SELECTED`.
6. If any branch times out after 60 seconds, the workflow short-circuits: it collects whichever variants returned and asks the Supervisor to select from those. The job enters `PARTIAL`.

A `JobSimulator` (TimedAction) drips a sample job every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TranslationSupervisor` | `AutonomousAgent` | Decomposes a job into variant instructions; later selects the best completed variant and runs the output guardrail. | `TranslationWorkflow` | returns typed result to workflow |
| `TranslationWorker` | `AutonomousAgent` | Produces one translation variant given a style instruction and source text. | `TranslationWorkflow` | — |
| `TranslationWorkflow` | `Workflow` | Coordinates the parallel fan-out to N worker branches, collects variants, calls supervisor for selection, runs guardrail. | `TranslationEndpoint`, `JobQueueConsumer` | `TranslationJobEntity` |
| `TranslationJobEntity` | `EventSourcedEntity` | Holds the job's lifecycle (pending → in-progress → selected / partial / blocked). | `TranslationWorkflow` | `TranslationView` |
| `JobQueue` | `EventSourcedEntity` | Logs each submitted job for replay/audit. | `TranslationEndpoint`, `JobSimulator` | `JobQueueConsumer` |
| `TranslationView` | `View` | List-of-jobs read model. | `TranslationJobEntity` events | `TranslationEndpoint` |
| `JobQueueConsumer` | `Consumer` | Listens to `JobQueue` events and starts a workflow per submission. | `JobQueue` events | `TranslationWorkflow` |
| `JobSimulator` | `TimedAction` | Drips a sample job every 60 s. | scheduler | `JobQueue` |
| `EvalSampler` | `TimedAction` | Samples one selected job every 5 minutes for eval scoring; emits a `SelectionEvalScored` event. | scheduler | `TranslationJobEntity` |
| `TranslationEndpoint` | `HttpEndpoint` | `/api/translation/*` — submit, get, list, SSE. | — | `TranslationView`, `JobQueue`, `TranslationJobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record JobRequest(String sourceText, String targetLanguage, String requestedBy) {}

record VariantInstruction(String register, String instruction) {}
record VariantPlan(List<VariantInstruction> instructions) {}

record TranslationVariant(String register, String translatedText, Instant producedAt) {}

record SelectionResult(
    TranslationVariant selectedVariant,
    String selectionRationale,
    String guardrailVerdict,
    Instant selectedAt
) {}

record TranslationJob(
    String jobId,
    String sourceText,
    String targetLanguage,
    JobStatus status,
    Optional<List<TranslationVariant>> variants,
    Optional<SelectionResult> selection,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { PENDING, IN_PROGRESS, SELECTED, PARTIAL, BLOCKED }
```

### Events (on `TranslationJobEntity`)

`JobCreated`, `BranchesStarted`, `VariantCollected`, `JobSelected`, `JobPartial`, `JobBlocked`, `SelectionEvalScored`.

### Events (on `JobQueue`)

`JobSubmitted { jobId, sourceText, targetLanguage, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/translation` — body `{ sourceText, targetLanguage }` → `{ jobId }`. Starts a workflow.
- `GET /api/translation` — list all jobs. Optional `?status=PENDING|IN_PROGRESS|SELECTED|PARTIAL|BLOCKED`.
- `GET /api/translation/{id}` — one job.
- `GET /api/translation/sse` — server-sent events stream of every job change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Parallel Best-of-N"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a source text and target language, live list of jobs with status pills, expand-row to see all variants + selected result + eval score.

Browser title: `<title>Akka Sample: Parallel Best-of-N</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `TranslationSupervisor`): vets the selected translation for policy violations and quality floor before it is returned. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one selected job every 5 minutes and emits a `SelectionEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `TranslationSupervisor` → `prompts/supervisor.md`. Decomposes the job into variant instructions; later selects the best variant.
- `TranslationWorker` → `prompts/worker.md`. Produces one translation variant for a given style instruction.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a source text; job progresses PENDING → IN_PROGRESS → SELECTED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set one branch timeout to 1 s); job enters PARTIAL with the remaining variants; supervisor selects from those.
3. **J3** — Inject a guardrail failure (Supervisor returns a flagged translation); job enters BLOCKED.
4. **J4** — Wait after a successful selection; the job's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-parallel-best-of-n demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-akka-parallel-best-of-n.
Java package io.akka.samples.openaiagentsparallelizationpattern. Akka 3.6.0. HTTP port 9599.

Components to wire (exactly):
- 2 AutonomousAgents:
  * TranslationSupervisor — definition() with
    capability(TaskAcceptance.of(PLAN_VARIANTS).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SELECT_BEST).maxIterationsPerTask(3)).
    System prompt loaded from prompts/supervisor.md.
    Returns VariantPlan{instructions: List<VariantInstruction{register, instruction}>}
    for PLAN_VARIANTS and SelectionResult{selectedVariant, selectionRationale,
    guardrailVerdict, selectedAt} for SELECT_BEST.
  * TranslationWorker — capability(TaskAcceptance.of(TRANSLATE).maxIterationsPerTask(3)).
    System prompt from prompts/worker.md.
    Returns TranslationVariant{register, translatedText, producedAt}.

- 1 Workflow TranslationWorkflow with steps:
  planStep -> [parallel] branch1, branch2, branch3 -> joinStep -> selectStep ->
  guardrailStep -> emitStep.
  planStep calls forAutonomousAgent(TranslationSupervisor.class, PLAN_VARIANTS).
  branch1, branch2, branch3 each call forAutonomousAgent(TranslationWorker.class, TRANSLATE)
  with a different VariantInstruction from the plan. All three run via CompletionStage
  allOf (or zip chaining); each governed by
  WorkflowSettings.builder().stepTimeout(TranslationWorkflow::branch1Step, ofSeconds(60)),
  stepTimeout(TranslationWorkflow::branch2Step, ofSeconds(60)),
  stepTimeout(TranslationWorkflow::branch3Step, ofSeconds(60)). On any branch timeout,
  transition to a collectPartialStep that collects whichever variants returned, then
  continues to selectStep with those. selectStep calls
  forAutonomousAgent(TranslationSupervisor.class, SELECT_BEST) with the collected variants;
  give selectStep a 90s stepTimeout. guardrailStep runs a deterministic content vetter +
  LLM judge on the selected translation; on failure, end with JobBlocked.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TranslationJobEntity holding state TranslationJob{jobId, sourceText,
  targetLanguage, JobStatus, Optional<List<TranslationVariant>> variants,
  Optional<SelectionResult> selection, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. JobStatus enum: PENDING, IN_PROGRESS, SELECTED, PARTIAL,
  BLOCKED. Events: JobCreated, BranchesStarted, VariantCollected, JobSelected, JobPartial,
  JobBlocked, SelectionEvalScored. Commands: createJob, startBranches, collectVariant,
  selectJob, partialJob, blockJob, recordEval, getJob.
  emptyState() returns TranslationJob.initial("", null, null) with no commandContext() reference.

- 1 EventSourcedEntity JobQueue with command enqueueJob(sourceText, targetLanguage,
  requestedBy) emitting JobSubmitted{jobId, sourceText, targetLanguage, requestedBy,
  submittedAt}.

- 1 View TranslationView with row type TranslationJobRow (mirrors TranslationJob minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  TranslationJobEntity events. ONE query getAllJobs SELECT * AS jobs FROM translation_view.
  No WHERE status filter — caller filters client-side.

- 1 Consumer JobQueueConsumer subscribed to JobQueue events; on JobSubmitted starts a
  TranslationWorkflow with the jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 60s, reads next line from
    src/main/resources/sample-events/translation-jobs.jsonl and calls JobQueue.enqueueJob.
  * EvalSampler — every 5 minutes, queries TranslationView.getAllJobs, picks the oldest
    SELECTED job without an evalScore, runs a 1–5 rubric judge over the selection content,
    then calls TranslationJobEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * TranslationEndpoint at /api with POST /translation, GET /translation,
    GET /translation/{id}, GET /translation/sse, and the /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TranslationTasks.java declaring three Task<R> constants: PLAN_VARIANTS (VariantPlan),
  TRANSLATE (TranslationVariant), SELECT_BEST (SelectionResult).
- Domain records JobRequest, VariantInstruction, VariantPlan, TranslationVariant,
  SelectionResult.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9599 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/translation-jobs.jsonl with 8 canned job lines
  (sourceText + targetLanguage pairs covering English→Spanish, English→French,
  English→German, English→Japanese, and English→Portuguese).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = general content-generation,
  decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/worker.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Parallel Best-of-N", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no
  ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Parallel Best-of-N</title>. No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (supervisor.json,
  worker.json), picks one entry pseudo-randomly per call, and deserialises
  it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of either VariantPlan or SelectionResult objects.
      4–6 VariantPlan entries (each with 3 VariantInstruction items: formal,
      informal, literal registers) and 4–6 SelectionResult entries (each
      selecting a variant with a 2–4 sentence rationale,
      guardrailVerdict = "ok").
    worker.json — 6–9 TranslationVariant entries covering the three register
      styles (formal, informal, literal) with 1–3 sentence translatedText
      per variant in plausible target-language text (or placeholder text when
      the mock language is unknown).
- A MockModelProvider.seedFor(jobId) helper makes the selection deterministic
  per job id so the same job produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s branches,
  90s selection); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion TranslationTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9599 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf / zip, NOT sequential calls.
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
