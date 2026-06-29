# SPEC — spec-to-pr

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Spec to Implementation PR.
**One-line pitch:** A user submits a product spec; one `ImplementationAgent` walks it through three task phases — **PARSE** the spec into structured requirements, **PLAN** the code changes needed, **DRAFT** a pull request grounded in the codebase — with each phase gated on the prior phase's recorded output and a mandatory human review hold before the PR is merge-eligible.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a developer-tooling domain. One `ImplementationAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the PARSE task's typed output becomes the PLAN task's instruction context; the PLAN task's typed output becomes the DRAFT task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Three governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`PARSE` / `PLAN` / `DRAFT`) and the current `SpecRunEntity` status. A DRAFT-phase tool (one that touches the mock repository) called while the entity has not yet recorded `PlanProduced` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget.
- A **`human-in-the-loop` hold** sits between `PrDrafted` and any merge-eligible state. After the DRAFT phase completes, the workflow enters `AWAITING_REVIEW` and parks. A human reviewer calls `POST /api/runs/{id}/approve` or `POST /api/runs/{id}/reject`. Only an explicit approve signal advances the run to CI.
- A **`ci-gate`** runs after `ReviewApproved`. `CiScorer` simulates a test suite against the draft PR's change plan and emits a pass/fail verdict. Only `CI_PASSED` advances the run to `MERGE_READY`.

The blueprint shows that a sequential pipeline for code generation is not just a chain of LLM calls — the HITL hold and CI gate are explicit checkpoints that prevent automated writes from reaching production without human oversight and automated quality checks.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a **product spec** into the input (or picks one of three seeded specs — `Add rate-limiting middleware`, `Refactor auth module to JWT`, `Add OpenTelemetry tracing to service layer`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `PARSING` — the workflow has started `parseStep` and the agent has been handed the PARSE task.
4. Within ~10–20 s the card reaches `PLANNING` — the typed `ParsedSpec` is visible in the card detail (a requirements table with each requirement, its priority, and affected files). The agent's PARSE task returned; the workflow recorded `SpecParsed` and ran the PLAN task.
5. Within ~10–20 s more the card reaches `DRAFTING`. The `ChangePlan` is visible (a list of file edits with filePath, changeType, and rationale).
6. Within ~10–20 s more the card reaches `AWAITING_REVIEW`. The right pane shows the full typed `DraftPr` — title, description, per-file `FileChange` list with proposed diffs — plus a reviewer action bar with **Approve** and **Reject** buttons.
7. After the reviewer clicks **Approve**, the run enters `CI_RUNNING`. Within ~2–5 s it reaches either `CI_PASSED` → `MERGE_READY` or `CI_FAILED` (blocked).
8. The user can submit another spec; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SpecEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE, approve/reject; serves `/api/metadata/*`. | — | `SpecRunEntity`, `SpecRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SpecRunEntity` | `EventSourcedEntity` | Per-run lifecycle: created → parsing → parsed → planning → planned → drafting → drafted → awaiting-review → ci-running → ci-passed → merge-ready / ci-failed / rejected. Source of truth. | `SpecEndpoint`, `SpecPipelineWorkflow` | `SpecRunView` |
| `SpecPipelineWorkflow` | `Workflow` | One workflow per runId. Steps: `parseStep` → `planStep` → `draftStep` → `reviewHoldStep` → `ciCheckStep`. Each LLM step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. `reviewHoldStep` parks until a human signal arrives. | started by `SpecEndpoint` after `CREATED` | `ImplementationAgent`, `SpecRunEntity`, `CiScorer` |
| `ImplementationAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `SpecTasks.java`: `PARSE_SPEC` → `ParsedSpec`, `PLAN_CHANGES` → `ChangePlan`, `DRAFT_PR` → `DraftPr`. Each task is registered with the phase-appropriate function tools. | invoked by `SpecPipelineWorkflow` | returns typed results |
| `ParseTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `extractRequirements(specText)` and `identifyAffectedFiles(requirements)`. Reads from `src/main/resources/sample-data/codebase/*.json` for deterministic offline output. | called from PARSE task | returns `List<Requirement>` / `List<String>` |
| `PlanTools` | function-tools class | Implements `lookupFileContent(filePath)` and `proposeEdit(filePath, description)`. Pure in-memory transformations against the mock codebase. | called from PLAN task | returns `String` / `FileChange` |
| `DraftTools` | function-tools class | Implements `composePrTitle(plan)` and `composePrDescription(spec, plan)`. Assembles the PR metadata from the preceding typed outputs. | called from DRAFT task | returns `String` |
| `WriteGuardrail` | `before-tool-call` guardrail (registered on `ImplementationAgent`) | Reads the in-flight task's declared phase and the current `SpecRunEntity` status. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `CiScorer` | plain class (no Akka primitive) | Pure deterministic CI simulator. Inputs: `DraftPr`, `ChangePlan`. Output: `CiResult{passed, checks}`. | called from `ciCheckStep` | returns result |
| `SpecRunView` | `View` | Read model: one row per run for the UI. | `SpecRunEntity` events | `SpecEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Requirement(String reqId, String text, String priority, List<String> affectedFiles) {}

record ParsedSpec(String specTitle, List<Requirement> requirements, Instant parsedAt) {}

record FileChange(String filePath, String changeType, String rationale, String proposedDiff) {}

record ChangePlan(List<FileChange> changes, Instant plannedAt) {}

record DraftPr(
    String title,
    String description,
    List<FileChange> fileChanges,
    Instant draftedAt
) {}

record CiCheck(String name, boolean passed, String message) {}

record CiResult(
    boolean passed,
    List<CiCheck> checks,
    Instant evaluatedAt
) {}

record ReviewDecision(
    String reviewerId,
    String decision,    // "approved" | "rejected"
    String comment,
    Instant decidedAt
) {}

record GuardrailRejection(
    String phase,
    String tool,
    String reason,
    Instant rejectedAt
) {}

record SpecRunRecord(
    String runId,
    Optional<String> specText,
    Optional<ParsedSpec> parsedSpec,
    Optional<ChangePlan> changePlan,
    Optional<DraftPr> draftPr,
    Optional<ReviewDecision> reviewDecision,
    Optional<CiResult> ciResult,
    SpecRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<GuardrailRejection> guardrailRejections
) {}

enum SpecRunStatus {
    CREATED, PARSING, PARSED, PLANNING, PLANNED,
    DRAFTING, DRAFTED, AWAITING_REVIEW, CI_RUNNING,
    CI_PASSED, MERGE_READY, CI_FAILED, REJECTED, FAILED
}
```

Events on `SpecRunEntity`: `RunCreated`, `ParseStarted`, `SpecParsed`, `PlanStarted`, `PlanProduced`, `DraftStarted`, `PrDrafted`, `ReviewApproved`, `ReviewRejected`, `CiStarted`, `CiPassed`, `CiFailed`, `GuardrailRejected`, `RunFailed`.

Every nullable lifecycle field on the `SpecRunRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ specText }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `POST /api/runs/{id}/approve` — body `{ reviewerId, comment }` → `204`.
- `POST /api/runs/{id}/reject` — body `{ reviewerId, comment }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Spec to Implementation PR</title>`.

The App UI tab is a two-column layout: a left rail with the live list of runs (status pill + spec title + age) and a right pane with the selected run's detail — spec text, parsed requirements table, change plan file list, draft PR (title, description, per-file diff), CI check list, reviewer action bar (Approve/Reject buttons, only active in `AWAITING_REVIEW`), eval result, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (phase-gate)**: `WriteGuardrail` is registered on `ImplementationAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.PARSE`, `Phase.PLAN`, `Phase.DRAFT`) and the current `SpecRunEntity.status` for the run the task is bound to. The accept rule is precise: `PARSE` tools require `status ∈ {CREATED, PARSING}`; `PLAN` tools require `status ∈ {PARSED, PLANNING}` AND `parsedSpec.isPresent()`; `DRAFT` tools require `status ∈ {PLANNED, DRAFTING}` AND `changePlan.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility.
- **H1 — HITL application hold**: `reviewHoldStep` parks the workflow after `PrDrafted`. The workflow timer is indefinite (no auto-advance). A human reviewer calls `POST /api/runs/{id}/approve` or `POST /api/runs/{id}/reject`. Only `ReviewApproved` triggers `ciCheckStep`. `ReviewRejected` transitions the run to `REJECTED` (terminal). The reviewer action bar in the App UI is the primary surface; direct API calls are also accepted.
- **C1 — CI test-gate**: `ciCheckStep` runs `CiScorer` over `(draftPr, changePlan)` immediately after `ReviewApproved`. The scorer is deterministic and rule-based (no LLM call — keeping the single-agent invariant honest). Three checks: compile check (all referenced filePaths are present in the mock codebase manifest), test check (no `changeType = "delete"` on a file that has a paired test), and lint check (no `proposedDiff` longer than 200 lines). Each check yields one `CiCheck{name, passed, message}`. `CiResult.passed = checks.stream().allMatch(c -> c.passed)`. On pass, emits `CiPassed` → status `MERGE_READY`. On failure, emits `CiFailed` → status `CI_FAILED` (blocked, no merge).

## 9. Agent prompts

- `ImplementationAgent` → `prompts/implementation-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded spec `Add rate-limiting middleware`; within 60 s the run reaches `AWAITING_REVIEW` with non-empty requirements, ≥ 1 file change, and a PR title on the card.
2. **J2** — The agent's first iteration on a run calls a DRAFT-phase tool (`composePrTitle`) before `PlanProduced` has been recorded (mock LLM path). `WriteGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the run eventually reaches `AWAITING_REVIEW` correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A reviewer calls `POST /api/runs/{id}/reject`; the run transitions to `REJECTED`; the reviewer action bar disappears; the card status shows `REJECTED` in red. No CI run is triggered.
4. **J4** — A reviewer calls `POST /api/runs/{id}/approve`; the run transitions to `CI_RUNNING`; the mock CI scorer returns a failing `test` check; the run transitions to `CI_FAILED`; the card is blocked from `MERGE_READY`. The CI check table in the right pane shows the failing check with its message.
5. **J5** — A run whose mock trajectory triggers `CiPassed` transitions to `MERGE_READY`. The card's status pill turns green and the reviewer action bar is replaced by a `MERGE_READY` badge.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named spec-to-pr demonstrating the sequential-pipeline x dev-code cell. Runs out
of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-dev-code-spec-to-pr. Java package io.akka.samples.spectoimplementationpr.
Akka 3.6.0. HTTP port 9745.

Components to wire (exactly):

- 1 AutonomousAgent ImplementationAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/implementation-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the PARSE, PLAN, and DRAFT tool sets are ALL registered on the agent;
  phase gating is the job of WriteGuardrail, NOT of conditional .tools(...) wiring. The
  before-tool-call guardrail (WriteGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow SpecPipelineWorkflow per runId with five steps:
  * parseStep — emits ParseStarted on the entity, then calls componentClient
    .forAutonomousAgent(ImplementationAgent.class, "agent-" + runId).runSingleTask(
      TaskDef.instructions("Spec text:\n" + specText + "\nPhase: PARSE\nExtract requirements
      and identify affected files.")
        .metadata("runId", runId)
        .metadata("phase", "PARSE")
        .taskType(SpecTasks.PARSE_SPEC)
    ). Reads forTask(taskId).result(PARSE_SPEC) to get ParsedSpec. Writes
    SpecRunEntity.recordParsedSpec(parsedSpec). WorkflowSettings.stepTimeout 60s.
  * planStep — emits PlanStarted, then runSingleTask with TaskDef.instructions
    (formatPlanContext(parsedSpec)) and metadata.phase = "PLAN", taskType PLAN_CHANGES.
    Writes SpecRunEntity.recordChangePlan(changePlan). stepTimeout 60s.
  * draftStep — emits DraftStarted, then runSingleTask with TaskDef.instructions
    (formatDraftContext(parsedSpec, changePlan)) and metadata.phase = "DRAFT", taskType
    DRAFT_PR. Writes SpecRunEntity.recordDraftPr(draftPr). stepTimeout 60s.
  * reviewHoldStep — emits nothing (entity is already in AWAITING_REVIEW after PrDrafted).
    The step suspends indefinitely using Workflow.pause() waiting for an external signal.
    When SpecRunEntity receives ReviewApproved or ReviewRejected (via the endpoint), the
    workflow is resumed via componentClient.forWorkflow(...).signal("review-decision",
    decision). stepTimeout omitted — hold is unbounded by design.
  * ciCheckStep — emits CiStarted, runs CiScorer.evaluate(draftPr, changePlan), writes
    CiPassed or CiFailed accordingly. stepTimeout 10s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(SpecPipelineWorkflow::error). The error step writes RunFailed
  and ends.

- 1 EventSourcedEntity SpecRunEntity (one per runId). State SpecRunRecord{runId,
  specText: Optional<String>, parsedSpec: Optional<ParsedSpec>, changePlan:
  Optional<ChangePlan>, draftPr: Optional<DraftPr>, reviewDecision: Optional<ReviewDecision>,
  ciResult: Optional<CiResult>, status: SpecRunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, guardrailRejections: List<GuardrailRejection>}.
  SpecRunStatus enum: CREATED, PARSING, PARSED, PLANNING, PLANNED, DRAFTING, DRAFTED,
  AWAITING_REVIEW, CI_RUNNING, CI_PASSED, MERGE_READY, CI_FAILED, REJECTED, FAILED.
  Events: RunCreated{specText}, ParseStarted, SpecParsed{parsedSpec}, PlanStarted,
  PlanProduced{changePlan}, DraftStarted, PrDrafted{draftPr}, ReviewApproved{reviewDecision},
  ReviewRejected{reviewDecision}, CiStarted, CiPassed{ciResult}, CiFailed{ciResult},
  GuardrailRejected{phase, tool, reason, rejectedAt}, RunFailed{reason}.
  Commands: create, startParse, recordParsedSpec, startPlan, recordChangePlan, startDraft,
  recordDraftPr, approve, reject, startCi, recordCiPassed, recordCiFailed,
  recordGuardrailRejection, fail, getRun. emptyState() returns SpecRunRecord.initial("")
  with all Optional fields as Optional.empty() and no commandContext() reference (Lesson 3).

- 1 View SpecRunView with row type SpecRunRow that mirrors SpecRunRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes SpecRunEntity events. ONE
  query getAllRuns: SELECT * AS runs FROM spec_run_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SpecEndpoint at /api with POST /runs (body {specText}; mints runId; calls
    SpecRunEntity.create(specText); then starts SpecPipelineWorkflow with id
    "pipeline-" + runId; returns {runId}), GET /runs (list from getAllRuns, sorted
    newest-first), GET /runs/{id} (one row), GET /runs/sse (Server-Sent Events forwarded
    from the view's stream-updates), POST /runs/{id}/approve (body {reviewerId, comment};
    calls SpecRunEntity.approve(...) and signals the workflow), POST /runs/{id}/reject
    (body {reviewerId, comment}; calls SpecRunEntity.reject(...) and signals the workflow),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- SpecTasks.java declaring three Task<R> constants:
    PARSE_SPEC = Task.name("Parse spec").description("Extract requirements and affected
      files from a product spec").resultConformsTo(ParsedSpec.class);
    PLAN_CHANGES = Task.name("Plan changes").description("Propose file edits that satisfy
      the parsed requirements").resultConformsTo(ChangePlan.class);
    DRAFT_PR = Task.name("Draft PR").description("Compose a pull-request title and
      description grounded in the change plan").resultConformsTo(DraftPr.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {PARSE, PLAN, DRAFT}. Each function-tool method carries the constant
  phase, e.g. @FunctionTool(name = "extractRequirements", phase = Phase.PARSE)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- ParseTools.java — @FunctionTool extractRequirements(String specText) -> List<Requirement>
  (one Requirement per bullet/sentence, with reqId minted as "r-" + sha1(text).substring(
  0,8), priority inferred from keywords); @FunctionTool identifyAffectedFiles(
  List<Requirement> requirements) -> List<String> (matched against
  src/main/resources/sample-data/codebase/file-manifest.json).

- PlanTools.java — @FunctionTool lookupFileContent(String filePath) -> String (reads from
  the mock codebase JSON keyed by filePath); @FunctionTool proposeEdit(String filePath,
  String description) -> FileChange (generates a deterministic stub diff).

- DraftTools.java — @FunctionTool composePrTitle(ChangePlan plan) -> String (derives a
  title from the dominant requirement text); @FunctionTool composePrDescription(
  ParsedSpec spec, ChangePlan plan) -> String (builds a Markdown description listing
  requirements and file changes).

- WriteGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the SpecRunEntity status by runId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("phase-violation: <tool> requires <precondition>,
  saw <status>"). On reject ALSO calls SpecRunEntity.recordGuardrailRejection(phase, tool,
  reason) so the rejection is visible in the UI's rejection-log strip.

- CiScorer.java — pure deterministic logic (no LLM). Inputs: DraftPr, ChangePlan.
  Outputs: CiResult with passed and checks list. Three checks:
  (1) compile — all FileChange.filePaths in changePlan are listed in file-manifest.json;
  (2) test — no FileChange with changeType = "delete" for a file that has a
  "_test" or "Test" counterpart in the manifest;
  (3) lint — no FileChange.proposedDiff longer than 200 lines.
  CiResult.passed = all three checks pass.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9745 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/specs.jsonl with 5 seeded spec lines covering the three
  surfaces named in J1–J5 plus two extras.

- src/main/resources/sample-data/codebase/*.json — three files (file-manifest.json with
  all known file paths; two per-spec signal files keyed by seeded spec slug) with
  deterministic content so ParseTools.extractRequirements returns the same list across
  restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, C1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list. No
  regulation_anchors — general dev-code domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (specs are
  code-level, not person-level), decisions.authority_level = recommend-only (the PR is
  advisory until a human approves), oversight.human_in_loop = true (reviewer must approve
  before CI and merge), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "phase-violation", "hallucinated-file",
  "stale-spec", "ci-test-regression"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/implementation-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Spec to Implementation PR",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with spec text, requirements table,
  change plan file list, draft PR, CI check table, reviewer action bar, guardrail
  rejection-log strip). Browser title exactly:
  <title>Akka Sample: Spec to Implementation PR</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
        Sets model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(runId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    parse-spec.json — 5 ParsedSpec entries. Each entry's tool_calls array contains
      1 extractRequirements(specText) + 1 identifyAffectedFiles(requirements) calls.
      Plus 1 deliberately PHASE-VIOLATING entry whose tool_calls array starts with
      composePrTitle(...) (a DRAFT-phase tool called during the PARSE phase) — the
      guardrail rejects it, the mock then falls through to a normal parse sequence. The
      mock should select the violating entry on the FIRST iteration of every 3rd run
      (modulo seed) so J2 is reproducible.
    plan-changes.json — 5 ChangePlan entries paired one-to-one with the parse entries,
      each with 2-4 FileChange items, with tool_calls containing lookupFileContent +
      proposeEdit in order.
    draft-pr.json — 5 DraftPr entries paired one-to-one. Each carries a title, description,
      and fileChanges list, tool_calls containing composePrTitle + composePrDescription.
      The CI scorer's failing test check will trigger for one entry (any FileChange with
      changeType = "delete" on a file with a Test counterpart) so J4 is reproducible.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ImplementationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SpecTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (parseStep
  60s, planStep 60s, draftStep 60s, ciCheckStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the SpecRunRecord row record is Optional<T>.
- Lesson 7: SpecTasks.java with PARSE_SPEC, PLAN_CHANGES, DRAFT_PR constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9745 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ImplementationAgent).
  CiScorer is deterministic and does NOT make an LLM call. ReviewHold does NOT make an LLM
  call — it is a pure workflow pause.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  WriteGuardrail is the runtime mechanism that enforces the phase order. Do NOT
  conditionally register tools per task.
- The HITL invariant: the workflow MUST NOT advance past AWAITING_REVIEW without an
  explicit human signal on the entity. Auto-approve after timeout is NOT permitted.
- The CI-gate invariant: CiScorer MUST run after every ReviewApproved. The gate is not
  optional — even a 3/3 pass must be explicit.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
