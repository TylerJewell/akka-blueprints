# SPEC — code-mode-sandbox

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Sandboxed Code-Execution Capability.
**One-line pitch:** Submit a coding task; a Planner decomposes it into sandboxed steps authorized by capability tokens, an Executor writes the code for each step, and the harness runs each step in a restricted in-process sandbox — blocking any step whose required scopes are not granted.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern where the executor's output (code) is run rather than returned as text. The Planner owns an **execution plan** (an ordered list of `SandboxStep` items; each step carries the source language, the code goal, and the required `CapabilityScope` set). The Executor writes the `CodePayload` for a single step. The harness then checks the capability registry, sanitizes the outbound prompt, runs the sandbox, and records the result.

The blueprint wires two governance mechanisms into that loop:

- a **before-tool-invocation guardrail** that checks a `CapabilityToken` before each sandbox call — the token controls which scopes (file I/O, network, subprocess, reflection) the step may exercise,
- a **prompt sanitizer** that strips secrets (API keys, JWTs, bearer tokens, high-entropy strings) from the prompt before it is sent to the Executor for code generation — preventing accidental embedding of credentials into generated code.

## 3. User-facing flows

The user opens the App UI tab and submits a coding task via the form.

1. The system creates an `ExecutionJob` in `QUEUED`, enqueues it in `JobQueue`, and starts an `ExecutionWorkflow`.
2. The Planner produces an `ExecutionPlan { steps: List<SandboxStep>, language, runtimeHints }` and emits `JobPlanned`. The job moves to `EXECUTING`.
3. The workflow enters the executor loop. Each iteration:
   - The workflow picks the next unexecuted `SandboxStep` from the plan.
   - The **prompt sanitizer** scrubs the step's `codeGoal` before sending it to the Executor.
   - The Executor produces a `CodePayload { sourceCode, entryPoint, expectedOutputSchema }`.
   - The **capability guardrail** vetoes the step if the `CodePayload`'s required scopes exceed the granted `CapabilityToken`; on veto the workflow records a `StepBlocked` entry and asks the Planner to revise.
   - The sandbox runs the `CodePayload` and returns a `StepResult { ok, stdout, stderr, exitCode }`.
   - The workflow appends a `StepRecord { step, payload, result, sanitizedGoal, recordedAt }` to the job's execution log.
4. After all steps complete, the Planner composes a `JobOutput { summary, artifacts }` and emits `JobCompleted`. The job moves to `COMPLETED`.
5. On three consecutive failures of the same step, or two consecutive replans without progress, the job moves to `FAILED`.
6. If the capability token is revoked mid-execution (operator action), the next step check blocks and the job moves to `BLOCKED`.

A `JobSimulator` (TimedAction) drips a sample coding task every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Decomposes a coding task into an `ExecutionPlan`. Replans on step failures or capability blocks. Composes the final `JobOutput`. | `ExecutionWorkflow` | returns typed result to workflow |
| `ExecutorAgent` | `AutonomousAgent` | Writes a `CodePayload` for a single `SandboxStep`. | `ExecutionWorkflow` | — |
| `ExecutionWorkflow` | `Workflow` | Drives the plan → sanitize-prompt → gate → execute → record → decide loop, replan branch, and terminal states. | `JobEndpoint`, `JobRequestConsumer` | `ExecutionJobEntity` |
| `ExecutionJobEntity` | `EventSourcedEntity` | Holds the job lifecycle, execution plan, step records, and final output. | `ExecutionWorkflow` | `ExecutionJobView` |
| `CapabilityRegistryEntity` | `EventSourcedEntity` | Holds the active `CapabilityToken` (scopes, issued-at, expires-at) for the deployment. Single instance keyed by `"default"`. | `JobEndpoint` (issue/revoke actions) | `ExecutionWorkflow` (gate reads) |
| `JobQueue` | `EventSourcedEntity` | Audit log of submitted jobs. | `JobEndpoint`, `JobSimulator` | `JobRequestConsumer` |
| `ExecutionJobView` | `View` | List-of-jobs read model for the UI. | `ExecutionJobEntity` events | `JobEndpoint` |
| `JobRequestConsumer` | `Consumer` | Subscribes to `JobQueue` events; starts an `ExecutionWorkflow` per submission. | `JobQueue` events | `ExecutionWorkflow` |
| `JobSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/job-prompts.jsonl` and enqueues it. | scheduler | `JobQueue` |
| `StaleJobMonitor` | `TimedAction` | Every 30 s, marks any job stuck in `EXECUTING` past 5 minutes as `STUCK`. The workflow polls this and ends with `JobFailedTimeout`. | scheduler | `ExecutionJobEntity` |
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE, capability issue/revoke. | — | `ExecutionJobView`, `JobQueue`, `ExecutionJobEntity`, `CapabilityRegistryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record JobRequest(String prompt, String requestedBy) {}

record SandboxStep(
    int ordinal,
    String codeGoal,
    String language,
    List<CapabilityScope> requiredScopes
) {}

record ExecutionPlan(
    List<SandboxStep> steps,
    String language,
    List<String> runtimeHints
) {}

record CodePayload(
    String sourceCode,
    String entryPoint,
    Optional<String> expectedOutputSchema
) {}

record StepResult(
    boolean ok,
    String stdout,
    String stderr,
    int exitCode
) {}

record StepRecord(
    int ordinal,
    SandboxStep step,
    String sanitizedGoal,
    Optional<CodePayload> payload,
    Optional<StepResult> result,
    StepVerdict verdict,
    Optional<String> blockReason,
    Instant recordedAt
) {}

record JobOutput(
    String summary,
    List<String> artifacts,
    Instant producedAt
) {}

record ExecutionJob(
    String jobId,
    String prompt,
    JobStatus status,
    Optional<ExecutionPlan> plan,
    List<StepRecord> stepRecords,
    Optional<JobOutput> output,
    Optional<String> failureReason,
    Optional<String> blockReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

record CapabilityToken(
    String tokenId,
    List<CapabilityScope> grantedScopes,
    Instant issuedAt,
    Optional<Instant> expiresAt,
    Optional<Instant> revokedAt
) {}

enum CapabilityScope { FILE_READ, FILE_WRITE, NETWORK, SUBPROCESS, REFLECTION }
enum StepVerdict { OK, BLOCKED_BY_CAPABILITY, FAILED, SANDBOX_ERROR }
enum JobStatus { QUEUED, PLANNING, EXECUTING, COMPLETED, FAILED, BLOCKED, STUCK }
```

### Events (`ExecutionJobEntity`)

`JobCreated`, `JobPlanned`, `StepStarted`, `StepBlocked`, `StepRecorded`, `PlanRevised`, `JobCompleted`, `JobFailed`, `JobBlocked`, `JobFailedTimeout`.

### Events (`CapabilityRegistryEntity`)

`TokenIssued`, `TokenRevoked`.

### Events (`JobQueue`)

`JobSubmitted { jobId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ prompt, requestedBy? }` → `202 { jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full plan + step records + output).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `POST /api/capability/issue` — body `{ scopes: List<CapabilityScope>, expiresInSeconds? }` → `200 CapabilityToken`. Issues or replaces the active token.
- `POST /api/capability/revoke` — `200`. Revokes the current token.
- `GET /api/capability` — `{ token?: CapabilityToken }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Sandboxed Code-Execution Capability"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a job, capability management panel, live list of jobs with status pills, expand-row to see the execution plan, the step records, and the final output.

Browser title: `<title>Akka Sample: Sandboxed Code-Execution Capability</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-invocation capability guardrail** (`before-tool-invocation` on `ExecutionWorkflow`): before each sandbox call, the workflow reads `CapabilityRegistryEntity.get("default")` and checks whether the `SandboxStep.requiredScopes` are a subset of the token's `grantedScopes` and whether the token is not expired or revoked. On failure the workflow records a `StepBlocked` entry and asks the Planner to revise the step within the granted scopes.
- **S1 — prompt sanitizer** (`sanitizer`, flavor `prompt`): the `codeGoal` field of every `SandboxStep` is scrubbed by `PromptSanitizer.scrub` before it is sent to `ExecutorAgent`. The sanitizer applies the same labelled-replacement regex set as `SecretScrubber` (AWS keys, GitHub tokens, JWTs, bearer tokens, high-entropy fallback). The sanitized goal is what appears in `StepRecord.sanitizedGoal` and in every subsequent Planner prompt.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Decomposes the task; decides next step.
- `ExecutorAgent` → `prompts/executor.md`. Writes code for a single step.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Write a Python script that reads numbers from stdin and prints their mean." Job progresses `QUEUED → PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows an execution plan with 2–5 steps, step records for each, and a non-empty `JobOutput`.
2. **J2** — Submit a task whose plan requires the `NETWORK` scope while the active capability token grants only `FILE_READ`. The guardrail blocks the affected step; the Planner replans within granted scopes or the job ends in `FAILED` with a clear reason. The blocked step never reaches the sandbox.
3. **J3** — Submit a task whose prompt contains a string matching `AKIA[0-9A-Z]{16}`. The sanitizer strips it from `codeGoal` before the Executor sees it; the `StepRecord.sanitizedGoal` contains `[REDACTED:aws-access-key]`; the generated `CodePayload.sourceCode` never contains the literal key pattern.
4. **J4** — Submit a task and revoke the capability token mid-execution (`POST /api/capability/revoke`). The next loop iteration's gate check blocks the pending step; the job transitions to `BLOCKED` with `blockReason = "capability token revoked"`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named code-mode-sandbox demonstrating the
planner-executor × dev-code cell. browser-or-sandbox-execution integration
(in-process restricted JVM sandbox; no Docker required).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-code-mode-sandbox.
Java package io.akka.samples.sandboxedcodeexecutioncapability. Akka 3.6.0.
HTTP port 9876.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(REVISE_PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_OUTPUT).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN returns ExecutionPlan.
    REVISE_PLAN takes (currentPlan, blockRecord) and returns a revised
    ExecutionPlan. COMPOSE_OUTPUT returns JobOutput.
  * ExecutorAgent — capability(TaskAcceptance.of(WRITE_CODE).maxIterationsPerTask(2)).
    Prompt from prompts/executor.md. Returns CodePayload.

- 1 Workflow ExecutionWorkflow with steps:
  planStep -> [loop entry] pickStepStep -> sanitizePromptStep ->
  gateStep -> executeStep -> recordStep -> decideStep
  -> [back to pickStepStep or to composeStep / failStep / blockedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), revisePlanStep ofSeconds(45),
    sanitizePromptStep ofSeconds(5), gateStep ofSeconds(10),
    executeStep ofSeconds(90) (covers sandbox run + executor call),
    decideStep ofSeconds(30), composeStep ofSeconds(60).
  defaultStepRecovery(maxRetries(2).failoverTo(ExecutionWorkflow::error)).
  sanitizePromptStep calls PromptSanitizer.scrub on the step's codeGoal; stores
    the sanitized goal in workflow state for use in gateStep and executeStep.
  gateStep reads CapabilityRegistryEntity.get("default"); checks that
    step.requiredScopes ⊆ token.grantedScopes and token is not revoked/expired.
    On gate fail: records a StepBlocked entry via ExecutionJobEntity.recordBlock
    and transitions to revisePlanStep.
  revisePlanStep calls PlannerAgent REVISE_PLAN; on success loops back to
    pickStepStep; on second consecutive revise-without-progress transitions
    to failStep.
  executeStep calls ExecutorAgent WRITE_CODE(sanitizedGoal, language,
    runtimeHints), then feeds the CodePayload to SandboxRunner.run(payload).
  recordStep calls ExecutionJobEntity.recordStep(stepRecord).
  decideStep: if more steps remain, loop; if all OK, composeStep; if step
    failed three times, failStep.
  composeStep calls PlannerAgent COMPOSE_OUTPUT; emits JobCompleted.
  blockedStep emits JobBlocked on ExecutionJobEntity.

- 1 EventSourcedEntity ExecutionJobEntity holding ExecutionJob state.
  emptyState() returns ExecutionJob.initial("", null). Commands:
  createJob, recordPlan, recordBlock, recordStep, revisePlan, completeJob,
  failJob, blockJob, timeoutFail, getJob. Events as listed in SPEC §5.

- 1 EventSourcedEntity CapabilityRegistryEntity keyed by literal "default".
  State CapabilityToken (nullable — no token issued yet). Commands:
  issueToken(scopes, expiresInSeconds), revokeToken, get. Events:
  TokenIssued, TokenRevoked.

- 1 EventSourcedEntity JobQueue with command enqueueJob(jobId, prompt,
  requestedBy) emitting JobSubmitted.

- 1 View ExecutionJobView with row type JobRow (mirror of ExecutionJob minus
  heavy step payloads — truncate to last 3 step records plus counts; the UI
  fetches the full job by id on click). Table updater consumes
  ExecutionJobEntity events. ONE query getAllJobs SELECT * AS jobs FROM
  execution_job_view. No WHERE status filter — caller filters client-side
  (Lesson 2).

- 1 Consumer JobRequestConsumer subscribed to JobQueue events; on JobSubmitted
  starts an ExecutionWorkflow with jobId as the workflow id.

- 2 TimedActions:
  * JobSimulator — every 90s, reads next line from
    src/main/resources/sample-events/job-prompts.jsonl and calls
    JobQueue.enqueueJob.
  * StaleJobMonitor — every 30s, queries ExecutionJobView.getAllJobs, filters
    EXECUTING jobs whose createdAt is older than 5 minutes, calls
    ExecutionJobEntity.timeoutFail; ExecutionWorkflow polls
    ExecutionJobEntity.getJob in its decideStep and exits when
    status == STUCK.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs, GET /jobs (filters client-side
    from getAllJobs), GET /jobs/{id}, GET /jobs/sse,
    POST /capability/issue, POST /capability/revoke, GET /capability, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo ExecutionPlan), REVISE_PLAN (ExecutionPlan),
  COMPOSE_OUTPUT (JobOutput).
- ExecutorTasks.java declaring one Task<R> constant: WRITE_CODE
  (resultConformsTo CodePayload).
- Domain records as listed in SPEC §5.
- application/PromptSanitizer.java — same regex/entropy patterns as
  SecretScrubber in the general template (AKIA..., gh token, JWT, sk-,
  Bearer, high-entropy ≥32 chars Shannon >4.5). Replacements tagged
  [REDACTED:aws-access-key] etc. Pure function; deterministic.
- application/CapabilityGuardrail.java — checks whether a step's
  requiredScopes are all present in the token's grantedScopes and whether
  the token has not been revoked or expired. Returns a GuardrailResult
  (PASS | BLOCK(reason)).
- application/SandboxRunner.java — deterministic in-process runner.
  Accepts CodePayload; looks up the matching entry in
  src/main/resources/sample-data/sandbox-results.jsonl by entryPoint hash;
  returns the canned StepResult. Throws SandboxException on unknown entry.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9876 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/job-prompts.jsonl with 8 canned coding
  task prompts (scripts, data transforms, file processors, string utils).
- src/main/resources/sample-data/sandbox-results.jsonl — 20 canned
  StepResult entries keyed by entryPoint hash. Include one entry whose
  stdout contains an AKIA-shaped key fragment to exercise the J3 sanitizer
  test (the sanitizer must catch it on the prompt side before the executor
  sees it).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, S1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance; deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/executor.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Sandboxed Code-Execution
  Capability", one-line pitch, prerequisites (including the integration
  form's host-software requirement: None), generate-the-system, what-you-
  get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form +
  capability management panel + live list with status pills and
  expand-on-click for plan, step records, and output). Browser title
  exactly: <title>Akka Sample: Sandboxed Code-Execution Capability</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM, (b) Name an existing env var, (c) Point to an existing
    env file, (d) Secrets-store URI, (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class
  name and Task<R> id. Each branch reads from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    planner.json — three lists keyed by task id:
      "PLAN" → 4 ExecutionPlan entries (2–5 steps each, spanning FILE_READ
      and SUBPROCESS scopes).
      "REVISE_PLAN" → 3 ExecutionPlan entries with reduced scope sets
      (FILE_READ only) so they pass the guardrail after a NETWORK block.
      "COMPOSE_OUTPUT" → 4 JobOutput entries with 60–120 word summaries
      and 2–4 artifact descriptions.
    executor.json — 8 CodePayload entries; one entry's sourceCode must
      include a substring matching "AKIAIOSFODNN7EXAMPLE" in a comment —
      to confirm the sanitizer scrubbed the prompt before this code was
      generated (the test checks the sanitizedGoal, not the code itself).

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every step that
  calls an agent or the sandbox.
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the ExecutionJob entity state.
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  ExecutorTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against provider current lineup.
  Defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9876 in application.conf.
- Lesson 11: source.platform never appears in user-facing surfaces.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string
  "browser-or-sandbox-execution" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
