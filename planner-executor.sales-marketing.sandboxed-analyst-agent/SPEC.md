# SPEC — sandboxed-analyst-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Sales Call Data Analyzer (Sandbox).
**One-line pitch:** Upload a sales call dataset; an AnalystAgent plans a sequence of analysis steps, submits each step as a Python script to a sandboxed execution environment, records the output on a progress ledger, and replans when steps fail — producing a structured `AnalysisReport` at completion.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern where the executor is a sandboxed code-execution environment rather than a hand-picked specialist agent. The AnalystAgent owns two ledgers — an **analysis ledger** (dataset facts, open hypotheses, plan, current execution step) and a **progress ledger** (each step's attempt count, verdict, code output). Each loop iteration the AnalystAgent reads both ledgers, proposes the next Python analysis script, and either continues, replans, halts, or completes. On three consecutive failures on the same step, or two consecutive replans without progress, the AnalystAgent emits a terminal failure.

The blueprint also demonstrates three governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vetoes each proposed script before it reaches the sandbox, blocking network access, filesystem writes outside `/workspace/`, use of `subprocess`, and shell escape patterns,
- an **automatic safety halt** that trips when the sandbox output signals an escape attempt or an unsafe-action signal,
- a **PII sanitizer** that scrubs phone numbers, email addresses, and name-shaped strings from every execution result before it lands on the progress ledger.

## 3. User-facing flows

The user opens the App UI tab and either uploads a CSV file or types a dataset description to simulate an upload.

1. The system creates an `AnalysisJob` record in `PLANNING` and starts an `AnalysisWorkflow`.
2. The AnalystAgent drafts an `AnalysisLedger { datasetFacts, hypotheses, plan, currentStep }` and emits `JobPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - AnalystAgent reads both ledgers and proposes a `StepDecision { scriptKind, pythonScript, rationale }`.
   - The **before-execution guardrail** vets the script; on rejection the workflow records a `StepBlocked` entry on the progress ledger and asks the AnalystAgent to revise.
   - `SandboxExecutorAgent` submits the script to the sandbox and returns a typed `ExecutionResult`.
   - The **PII sanitizer** scrubs the result's stdout.
   - The workflow appends a `ProgressEntry { stepKind, script, attempt, verdict, scrubbedOutput, blocker, recordedAt }` to the progress ledger.
   - The **automatic safety halt** evaluator inspects the new entry; on an unsafe-action signal it emits `JobHaltedAutomatic` and the workflow ends.
4. The AnalystAgent decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`. After three consecutive failures on the same step or two consecutive replans without progress, the AnalystAgent emits `FAIL`.
5. On `COMPLETE`, the AnalystAgent produces an `AnalysisReport { summary, findings, charts }` and emits `JobCompleted`. The job moves to `COMPLETED`.
6. The operator can press **Halt new steps** in the dashboard at any time. The workflow finishes the in-flight sandbox call, then ends with `JobHaltedOperator`. The job moves to `HALTED`.

A `DatasetSimulator` (TimedAction) drips a sample dataset upload every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AnalystAgent` | `AutonomousAgent` | Plans, decides, and composes the final report. Maintains analysis ledger; reads progress ledger. Produces `AnalysisReport` on completion. | `AnalysisWorkflow` | returns typed result to workflow |
| `SandboxExecutorAgent` | `AutonomousAgent` | Submits Python scripts to the E2B sandbox (or mock executor) and returns typed `ExecutionResult`. | `AnalysisWorkflow` | — |
| `AnalysisWorkflow` | `Workflow` | Drives the plan → check-halt → propose → guardrail → execute → sanitize → record → auto-halt-eval → decide loop, plus replan and halt branches. | `AnalysisEndpoint`, `UploadConsumer` | `AnalysisJobEntity` |
| `AnalysisJobEntity` | `EventSourcedEntity` | Holds the job's lifecycle, analysis ledger, progress ledger, and final report. | `AnalysisWorkflow` | `AnalysisJobView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `AnalysisEndpoint` (operator action) | `AnalysisWorkflow` (polls) |
| `UploadQueue` | `EventSourcedEntity` | Audit log of submitted dataset uploads. | `AnalysisEndpoint`, `DatasetSimulator` | `UploadConsumer` |
| `AnalysisJobView` | `View` | List-of-jobs read model for the UI. | `AnalysisJobEntity` events | `AnalysisEndpoint` |
| `UploadConsumer` | `Consumer` | Subscribes to `UploadQueue` events; starts an `AnalysisWorkflow` per upload. | `UploadQueue` events | `AnalysisWorkflow` |
| `DatasetSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/dataset-prompts.jsonl` and enqueues it. | scheduler | `UploadQueue` |
| `StaleJobMonitor` | `TimedAction` | Every 30 s, marks any job stuck in `EXECUTING` past 5 minutes as `STUCK`. The workflow polls this and ends with `JobFailedTimeout`. | scheduler | `AnalysisJobEntity` |
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/jobs/*` — upload, get, list, SSE, operator halt. | — | `AnalysisJobView`, `UploadQueue`, `AnalysisJobEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record DatasetUpload(String datasetName, String uploaderEmail, String csvContent) {}

record AnalysisLedger(
    List<String> datasetFacts,
    List<String> hypotheses,
    List<String> plan,
    Optional<StepDecision> currentStep
) {}

record StepDecision(
    ScriptKind scriptKind,
    String pythonScript,
    String rationale
) {}

record ExecutionResult(
    ScriptKind scriptKind,
    String script,
    boolean ok,
    String stdout,
    Optional<String> stderr
) {}

record ProgressEntry(
    int attempt,
    ScriptKind scriptKind,
    String script,
    ProgressVerdict verdict,
    String scrubbedOutput,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ProgressLedger(List<ProgressEntry> entries) {}

record AnalysisFinding(String metricName, String value, String interpretation) {}

record AnalysisReport(
    String summary,
    List<AnalysisFinding> findings,
    List<String> charts,
    Instant producedAt
) {}

record AnalysisJob(
    String jobId,
    String datasetName,
    String uploaderEmail,
    JobStatus status,
    Optional<AnalysisLedger> ledger,
    Optional<ProgressLedger> progress,
    Optional<AnalysisReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ScriptKind { LOAD_INSPECT, AGGREGATE, SENTIMENT, SEGMENT, VISUALISE, SUMMARISE }
enum ProgressVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum JobStatus { PLANNING, EXECUTING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`AnalysisJobEntity`)

`JobCreated`, `JobPlanned`, `StepDispatched`, `StepBlocked`, `StepRecorded`, `LedgerRevised`, `JobCompleted`, `JobFailed`, `JobHaltedAutomatic`, `JobHaltedOperator`, `JobFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`UploadQueue`)

`DatasetSubmitted { jobId, datasetName, uploaderEmail, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ datasetName, uploaderEmail?, csvContent? }` → `202 { jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=...`.
- `GET /api/jobs/{id}` — one job (full ledgers + report).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Sales Call Data Analyzer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to upload a dataset, operator halt/resume control, live list of jobs with status pills, expand-row to see the analysis ledger, the progress ledger entries, and the final report.

Browser title: `<title>Akka Sample: Sales Call Data Analyzer (Sandbox)</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-execution guardrail** (`before-tool-call` on `AnalystAgent`): every proposed `StepDecision.pythonScript` is checked against a script policy that forbids network access (`import requests`, `import urllib`, `import socket`), filesystem writes outside `/workspace/`, use of `subprocess` or `os.system`, and shell-escape patterns (backticks, `!` prefix). Blocking. Failure → `StepBlocked` entry + replan request.
- **HT1 — automatic safety halt** (`halt`, flavor `automatic-safety-halt`): after each `ProgressEntry` is appended, the workflow runs a deterministic evaluator over the entry's stdout and the script kind. On an unsafe-action signal (e.g., stdout contains sandbox escape evidence, forbidden-import import errors, or explicit prompt-injection markers), the workflow emits `JobHaltedAutomatic` and ends.
- **S1 — PII sanitizer** (`sanitizer`, flavor `pii`): every `ExecutionResult.stdout` is scrubbed by a deterministic redactor that matches phone numbers (E.164 and NANP patterns), email addresses, and name-shaped strings matching the dataset's known contact header columns before the entry is written to the progress ledger.

## 9. Agent prompts

- `AnalystAgent` → `prompts/analyst.md`. Maintains both ledgers; decides next step; composes final report.
- `SandboxExecutorAgent` → `prompts/sandbox-executor.md`. Receives a Python script and a dataset path; returns `ExecutionResult`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Upload a sales call CSV. Job progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows an analysis ledger with a non-empty plan, a progress ledger with 3–8 entries spanning at least `LOAD_INSPECT` and `AGGREGATE`, and a non-empty `AnalysisReport`.
2. **J2** — Upload a dataset and construct a prompt that would cause the analyst to propose a script importing `requests`. The guardrail blocks the dispatch on the first attempt; the analyst replans; the job either completes via an allowed path or fails after the replan budget is exhausted.
3. **J3** — Upload a dataset and click **Halt new steps** while the job is `EXECUTING`. The in-flight sandbox call finishes; no further steps execute; the job ends in `HALTED`.
4. **J4** — A fixture dataset contains a column of E.164 phone numbers. The progress ledger entry for the step that reads this data shows phone numbers replaced by `[REDACTED:phone]`; the AnalystAgent's next prompt never contains a literal phone number.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sandboxed-analyst-agent demonstrating the
planner-executor × sales-marketing cell. Integration tier: E2B sandbox
(E2B_API_KEY required; mock-execution path available).
Maven group io.akka.samples. Maven artifact planner-executor-sales-marketing-sandboxed-analyst-agent. Java
package io.akka.samples.salescalldataanalyzersandbox. Akka 3.6.0. HTTP port 9822.

Components to wire (exactly):
- 2 AutonomousAgents:
  * AnalystAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/analyst.md. PLAN returns AnalysisLedger.
    DECIDE returns a NextStep tagged union (Continue(StepDecision) |
    Replan(AnalysisLedger) | Complete(AnalysisReport stub) | Fail(String)).
    COMPOSE_REPORT returns AnalysisReport.
  * SandboxExecutorAgent — capability(TaskAcceptance.of(EXECUTE_SCRIPT).maxIterationsPerTask(2)).
    Prompt from prompts/sandbox-executor.md. Returns ExecutionResult.

- 1 Workflow AnalysisWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  executeStep -> sanitizeStep -> recordStep -> autoHaltEvalStep -> decideStep
  -> [back to checkHaltStep or to composeReportStep / completeStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), executeStep
    ofSeconds(120) (covers sandbox round-trip), decideStep ofSeconds(45),
    composeReportStep ofSeconds(60). defaultStepRecovery(maxRetries(2).failoverTo
    (AnalysisWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits JobHaltedOperator on AnalysisJobEntity).
  guardrailStep runs ScriptGuardrail.vet(StepDecision); on reject records a
  StepBlocked entry via AnalysisJobEntity.recordBlock(script, reason) and
  loops back to proposeStep.
  executeStep calls SandboxExecutorAgent via forAutonomousAgent(...).runSingleTask(...)
  then forTask(jobId).result(...).
  sanitizeStep applies PiiScrubber.scrub to the ExecutionResult.stdout.
  recordStep calls AnalysisJobEntity.recordProgress(entry).
  autoHaltEvalStep runs SafetyEvaluator.evaluate(entry) and, on unsafe,
  transitions to haltedStep (emits JobHaltedAutomatic).
  decideStep calls forAutonomousAgent(AnalystAgent.class, DECIDE);
  on Continue or Replan loops; on Complete transitions to composeReportStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity AnalysisJobEntity holding AnalysisJob state. emptyState()
  returns AnalysisJob.initial("", null) with no commandContext() reference. Commands:
  createJob, recordPlan, recordDispatch, recordBlock, recordProgress,
  reviseLedger, completeJob, failJob, haltAutomatic, haltOperator,
  timeoutFail, getJob. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity UploadQueue with command enqueueDataset(jobId, datasetName,
  uploaderEmail) emitting DatasetSubmitted.

- 1 View AnalysisJobView with row type AnalysisJobRow (mirror of AnalysisJob minus
  heavy ledger payloads — truncate to last 3 progress entries plus counts; the UI
  fetches the full job by id on click). Table updater consumes AnalysisJobEntity
  events. ONE query getAllJobs SELECT * AS jobs FROM analysis_job_view. No WHERE
  status filter — caller filters client-side (Lesson 2).

- 1 Consumer UploadConsumer subscribed to UploadQueue events; on DatasetSubmitted
  starts an AnalysisWorkflow with jobId as the workflow id.

- 2 TimedActions:
  * DatasetSimulator — every 90s, reads next line from
    src/main/resources/sample-events/dataset-prompts.jsonl and calls
    UploadQueue.enqueueDataset.
  * StaleJobMonitor — every 30s, queries AnalysisJobView.getAllJobs, filters
    EXECUTING jobs whose createdAt is older than 5 minutes, calls
    AnalysisJobEntity.timeoutFail; AnalysisWorkflow polls AnalysisJobEntity.getJob
    in its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /jobs, GET /jobs (filters client-side
    from getAllJobs), GET /jobs/{id}, GET /jobs/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- AnalystTasks.java declaring three Task<R> constants: PLAN
  (resultConformsTo AnalysisLedger), DECIDE (NextStep), COMPOSE_REPORT (AnalysisReport).
- ExecutorTasks.java declaring one Task<R> constant: EXECUTE_SCRIPT
  (resultConformsTo ExecutionResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits Continue, Replan, Complete, Fail (each carrying its own payload —
  Continue with StepDecision, Replan with revisedLedger, Complete with
  AnalysisReport stub, Fail with failureReason).
- application/PiiScrubber.java — deterministic regex scrubber.
  Patterns: E.164 phone (+1XXXXXXXXXX and variants), NANP (NXX-NXX-XXXX),
  RFC 5322 email addresses, name-header column values (first_name, last_name,
  contact_name patterns extracted from the dataset header row).
  Replacements: [REDACTED:phone], [REDACTED:email], [REDACTED:name].
- application/ScriptGuardrail.java — deterministic policy vetter. Reject if the
  script contains any of: import requests, import urllib, import socket,
  import subprocess, os.system, os.popen, shell=True, eval(, exec(,
  or backtick/shell-escape patterns. Also reject if the script references
  any path outside /workspace/ for write operations (open(.*,.*w) outside
  /workspace/).
- application/SafetyEvaluator.java — deterministic post-execution check.
  Flag UNSAFE on stdout containing "Permission denied" plus "subprocess",
  stdout containing sandbox-escape keywords ("__import__('os')",
  "os.getcwd() == '/'"), or stdout containing explicit prompt-injection
  markers ("ignore previous instructions", "<system>"). Otherwise OK.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9822 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also e2b.api-key = ${?E2B_API_KEY}.
- src/main/resources/sample-events/dataset-prompts.jsonl with 8 canned dataset
  upload descriptions representing different sales call CSV shapes.
- src/main/resources/sample-data/calls/*.csv — 6 short sales call CSV files
  used by SandboxExecutorAgent. Include one file whose content contains E.164
  phone numbers for the J4 acceptance test (the agent must surface them; the
  PII sanitizer must scrub them).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1, HT1, S1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, sector, data,
  decisions, failure, oversight, operations, and compliance; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/analyst.md, prompts/sandbox-executor.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Sales Call Data Analyzer
  (Sandbox)", one-line pitch, prerequisites (including integration tier host
  software: E2B + E2B_API_KEY), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (upload form + operator halt/resume control + live list with
  status pills and expand-on-click for ledgers and report). Browser title
  exactly: <title>Akka Sample: Sales Call Data Analyzer (Sandbox)</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class
  name and the Task<R> id.
- Per-agent mock-response shapes for THIS blueprint:
    analyst.json — three lists keyed by task id:
      "PLAN" → 4–6 AnalysisLedger entries (datasetFacts, hypotheses, plan
      steps covering LOAD_INSPECT, AGGREGATE, SENTIMENT, SEGMENT,
      VISUALISE, SUMMARISE).
      "DECIDE" → 4–6 NextStep entries covering Continue (with StepDecision
      across script kinds), Replan, Complete, Fail.
      "COMPOSE_REPORT" → 4–6 AnalysisReport entries with 60–120 word
      summaries and 3–5 findings.
    sandbox-executor.json — 6 ExecutionResult entries; ONE entry's stdout
      must include an E.164 phone string (+12025551234) so the J4 PII
      sanitizer test fires.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, executeStep,
  decideStep, composeReportStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the AnalysisJob entity state (ledger, progress, report, failureReason,
  haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion AnalystTasks.java and
  ExecutorTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9822 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "E2B + E2B_API_KEY" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow; no key
  value written to disk.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
