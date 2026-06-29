# SPEC — parallel-fanout

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Parallel Task Decomposition.
**One-line pitch:** Submit a job; a coordinator decomposes it into two concurrent subtasks, fans them out to specialist workers in parallel, then consolidates their outputs into one result.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their outputs, and asks a third AutonomousAgent to consolidate a unified result. The blueprint also demonstrates an **output validation** step that vets the consolidated result before it is persisted, and an **eval-event** that samples consolidation quality on a periodic schedule.

## 3. User-facing flows

The user opens the App UI tab and submits a job payload via the form.

1. The system creates a `TaskJob` record in `QUEUED` and starts a `TaskExecutionWorkflow`.
2. The coordinator decomposes the payload into two parallel work items: a structural-analysis subtask for SubtaskWorkerA, and a contextual-enrichment subtask for SubtaskWorkerB.
3. The workflow forks: both workers run concurrently. Each returns a typed payload.
4. The coordinator merges the two payloads into a `ConsolidatedResult { summary, structuralOutput, contextualOutput, validationVerdict }`.
5. An output validation step vets the consolidated result; if it fails, the job moves to `REJECTED`. Otherwise, the job moves to `CONSOLIDATED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the coordinator to consolidate from whichever side returned, and the job enters `DEGRADED`.

A `JobSimulator` (TimedAction) drips a sample job payload every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskCoordinator` | `AutonomousAgent` | Decomposes the job payload, consolidates merged worker outputs, runs output validation. | `TaskExecutionWorkflow` | returns typed result to workflow |
| `SubtaskWorkerA` | `AutonomousAgent` | Performs structural analysis on the job payload. Seeded tool returns canned results. | `TaskExecutionWorkflow` | — |
| `SubtaskWorkerB` | `AutonomousAgent` | Performs contextual enrichment on the job payload. Seeded tool returns canned results. | `TaskExecutionWorkflow` | — |
| `TaskExecutionWorkflow` | `Workflow` | Coordinates the parallel fan-out, the consolidation, the validation. | `TaskEndpoint`, `JobRequestConsumer` | `TaskJobEntity` |
| `TaskJobEntity` | `EventSourcedEntity` | Holds the job's lifecycle (queued → processing → consolidated / degraded / rejected). | `TaskExecutionWorkflow` | `TaskJobView` |
| `JobQueue` | `EventSourcedEntity` | Logs each submitted job for replay/audit. | `TaskEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `TaskJobView` | `View` | List-of-jobs read model. | `TaskJobEntity` events | `TaskEndpoint` |
| `JobRequestConsumer` | `Consumer` | Listens to `JobQueue` events and starts a workflow per submission. | `JobQueue` events | `TaskExecutionWorkflow` |
| `JobSimulator` | `TimedAction` | Drips a sample job payload every 60 s. | scheduler | `JobQueue` |
| `QualitySampler` | `TimedAction` | Samples one consolidated job every 5 minutes for quality scoring; emits a `QualityScored` event. | scheduler | `TaskJobEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `TaskJobView`, `JobQueue`, `TaskJobEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record JobRequest(String payload, String requestedBy) {}

record StructuralOutput(List<StructuralElement> elements, Instant processedAt) {}
record StructuralElement(String label, String category, String detail) {}

record ContextualOutput(String enrichedContext, List<String> tags, Instant enrichedAt) {}

record DecompositionPlan(String structuralQuery, String contextualQuery) {}

record ConsolidatedResult(String summary, StructuralOutput structural, ContextualOutput contextual,
                          String validationVerdict, Instant consolidatedAt) {}

record TaskJob(
    String jobId,
    String payload,
    JobStatus status,
    Optional<StructuralOutput> structural,
    Optional<ContextualOutput> contextual,
    Optional<ConsolidatedResult> consolidated,
    Optional<String> failureReason,
    Optional<Integer> qualityScore,
    Optional<String> qualityRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { QUEUED, PROCESSING, CONSOLIDATED, DEGRADED, REJECTED }
```

### Events (on `TaskJobEntity`)

`JobCreated`, `StructuralOutputAttached`, `ContextualOutputAttached`, `JobConsolidated`, `JobDegraded`, `JobRejected`, `QualityScored`.

### Events (on `JobQueue`)

`JobSubmitted { jobId, payload, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ payload }` → `{ jobId }`. Starts a workflow.
- `GET /api/tasks` — list all jobs. Optional `?status=QUEUED|PROCESSING|CONSOLIDATED|DEGRADED|REJECTED`.
- `GET /api/tasks/{id}` — one job.
- `GET /api/tasks/sse` — server-sent events stream of every job change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Parallel Task Decomposition"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a job payload, live list of jobs with status pills, expand-row to see structural output + contextual output + consolidated summary + quality score.

Browser title: `<title>Akka Sample: Parallel Task Decomposition</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires no controls for this baseline (controls list is empty). The eval-matrix.yaml and simplified_view are generated with an empty controls array.

## 9. Agent prompts

- `TaskCoordinator` → `prompts/coordinator.md`. Decomposes the job payload into work items; later consolidates results into the final output.
- `SubtaskWorkerA` → `prompts/worker-a.md`. Performs structural analysis; returns `StructuralOutput`.
- `SubtaskWorkerB` → `prompts/worker-b.md`. Performs contextual enrichment; returns `ContextualOutput`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a job payload; job progresses QUEUED → PROCESSING → CONSOLIDATED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `SubtaskWorkerA` timeout to 1 s); job enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a validation failure (coordinator returns invalid content); job enters REJECTED.
4. **J4** — Wait after a successful consolidation; the job's row in the UI shows a quality score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named parallel-fanout demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-parallel-fanout.
Java package io.akka.samples.paralleltaskdecompositionexecution. Akka 3.6.0. HTTP port 9386.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TaskCoordinator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/coordinator.md. Returns DecompositionPlan{structuralQuery, contextualQuery} for
    DECOMPOSE and ConsolidatedResult{summary, structural, contextual, validationVerdict, consolidatedAt}
    for CONSOLIDATE.
  * SubtaskWorkerA — capability(TaskAcceptance.of(ANALYSE_STRUCTURE).maxIterationsPerTask(3)).
    System prompt from prompts/worker-a.md. Returns
    StructuralOutput{elements: List<StructuralElement{label, category, detail}>, processedAt}.
  * SubtaskWorkerB — capability(TaskAcceptance.of(ENRICH_CONTEXT).maxIterationsPerTask(2)).
    System prompt from prompts/worker-b.md. Returns
    ContextualOutput{enrichedContext, tags: List<String>, enrichedAt}.

- 1 Workflow TaskExecutionWorkflow with steps:
  decomposeStep -> [parallel] structuralStep, contextualStep -> joinStep -> consolidateStep ->
  validateStep -> emitStep.
  decomposeStep calls forAutonomousAgent(TaskCoordinator.class, DECOMPOSE).
  structuralStep and contextualStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(TaskExecutionWorkflow::structuralStep, ofSeconds(60)) and
  stepTimeout(TaskExecutionWorkflow::contextualStep, ofSeconds(60)). On either timeout, transition to
  a degradeStep that calls consolidateStep with whichever side returned, then ends with JobDegraded.
  consolidateStep calls forAutonomousAgent(TaskCoordinator.class, CONSOLIDATE) with the merged
  inputs; give consolidateStep a 90s stepTimeout. validateStep runs deterministic schema validation
  plus an LLM judge on the consolidated content; on failure, end with JobRejected. WorkflowSettings
  is nested inside Workflow — no import.

- 1 EventSourcedEntity TaskJobEntity holding state TaskJob{jobId, payload,
  JobStatus, Optional<StructuralOutput> structural, Optional<ContextualOutput> contextual,
  Optional<ConsolidatedResult> consolidated, Optional<String> failureReason,
  Optional<Integer> qualityScore, Optional<String> qualityRationale, Instant createdAt,
  Optional<Instant> finishedAt}. JobStatus enum: QUEUED, PROCESSING, CONSOLIDATED,
  DEGRADED, REJECTED. Events: JobCreated, StructuralOutputAttached, ContextualOutputAttached,
  JobConsolidated, JobDegraded, JobRejected, QualityScored. Commands: createJob,
  attachStructural, attachContextual, consolidate, degrade, reject, recordQuality, getJob.
  emptyState() returns TaskJob.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity JobQueue with command submitJob(payload, requestedBy) emitting
  JobSubmitted{jobId, payload, requestedBy, submittedAt}.

- 1 View TaskJobView with row type TaskJobRow (mirrors TaskJob minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes TaskJobEntity events. ONE query
  getAllJobs SELECT * AS jobs FROM task_job_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer JobRequestConsumer subscribed to JobQueue events; on JobSubmitted starts a
  TaskExecutionWorkflow with the jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-jobs.jsonl and calls JobQueue.submitJob.
  * QualitySampler — every 5 minutes, queries TaskJobView.getAllJobs, picks the oldest
    CONSOLIDATED job without a qualityScore, runs a 1–5 rubric judge over the consolidated
    content, then calls TaskJobEntity.recordQuality(score, rationale).

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- TaskTasks.java declaring four Task<R> constants: DECOMPOSE (DecompositionPlan),
  ANALYSE_STRUCTURE (StructuralOutput), ENRICH_CONTEXT (ContextualOutput),
  CONSOLIDATE (ConsolidatedResult).
- Domain records DecompositionPlan, StructuralElement, StructuralOutput, ContextualOutput,
  ConsolidatedResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9386 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-jobs.jsonl with 8 canned job payload lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 0 controls and an empty simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = task-decomposition,
  decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/worker-a.md, prompts/worker-b.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Parallel Task Decomposition", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Parallel Task Decomposition</title>. No
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
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  worker-a.json, worker-b.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either DecompositionPlan or ConsolidatedResult objects.
      4–6 DecompositionPlan entries (structuralQuery + contextualQuery pairs) and
      4–6 ConsolidatedResult entries (each with a 60–120 word summary, a 2–5
      element structural output, a 3–6 tag contextual output,
      validationVerdict = "ok").
    worker-a.json — 4–6 StructuralOutput entries, each with 2–5 StructuralElements
      whose category values are drawn from: "component", "dependency", "interface",
      "constraint", "resource".
    worker-b.json — 4–6 ContextualOutput entries, each with a 40–80 word
      enrichedContext and 3–6 tags.
- A MockModelProvider.seedFor(jobId) helper makes the selection deterministic
  per job id so the same job produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s consolidation); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion TaskTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9386 in application.conf.
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
