# SPEC — code-interpreter-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CodeInterpreterAgent.
**One-line pitch:** A user submits a natural-language prompt and an optional data payload; one AI agent generates Python code to answer the prompt, the code runs in a sandboxed in-process executor, and a structured `ExecutionResult` — answer value, row table, or error — is returned through the workflow and surfaced in the UI.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `CodeInterpreterAgent` (AutonomousAgent) carries the entire decision of what code to write; the surrounding components only prepare input, gate execution, and record output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts the agent's generated code before the interpreter runs it. It inspects the code for forbidden module imports, shell-escape attempts, file-system writes outside the scratch space, and network calls. Code that fails any check is rejected and returned to the agent as a structured error so it rewrites and retries within the same task.
- An **automatic safety halt** bounds every execution by wall-clock time and estimated memory allocation. If the executor detects a runaway condition (execution exceeding the configured budget), it terminates the execution, records a `JobHalted` event, and transitions the entity to `HALTED`. No retry — the halt is a terminal safety signal.

The blueprint shows that a code-generating agent running untrusted computations requires pre-execution inspection as well as runtime enforcement; the two controls address different failure modes.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a natural-language **prompt** (e.g., "Find the top 5 values in the revenue column and compute their mean.") and optionally uploads or pastes a **data payload** (CSV text, JSON array, or raw numeric list).
2. The user picks a **dataset format** from a dropdown (`CSV`, `JSON`, `Numeric`) or pastes a custom payload, then clicks **Run**. The UI POSTs to `/api/jobs` and receives a `jobId`.
3. The job card appears in the live list in `SUBMITTED` state. Within ~1 s, the workflow's `guardStep` completes — the agent generates code and the guardrail either approves it (card moves to `CODE_APPROVED`) or rejects it (the agent retries internally).
4. Within ~5–20 s, `executeStep` finishes. The card transitions to `EXECUTING` then `RESULT_RECORDED`. The result appears: an `answer` string (for scalar outputs), an optional `rows` table (for tabular outputs), and the actual Python code that ran.
5. The user can submit another prompt; the live list keeps the history visible with age and result-type chips.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `JobEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `JobEntity`, `JobView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `JobEntity` | `EventSourcedEntity` | Per-job lifecycle: submitted → code-approved → executing → result-recorded → halted. Source of truth. | `JobEndpoint`, `InterpretationWorkflow` | `JobView` |
| `InterpretationWorkflow` | `Workflow` | One workflow per job. Steps: `guardStep` → `executeStep`. | started by `JobEndpoint` after entity creation | `CodeInterpreterAgent`, `ExecutionSandbox`, `JobEntity` |
| `CodeInterpreterAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user prompt and the data payload as a task attachment; emits a `code-execution` tool call with the Python code to run. Receives the execution result and returns `ExecutionResult`. | invoked by `InterpretationWorkflow` | returns result |
| `CodeGuardrail` | guardrail (supporting class) | `before-tool-call` hook on `CodeInterpreterAgent`. Inspects generated Python code; rejects on forbidden imports, shell escapes, file-system writes outside scratch space, or network calls. | wired to `CodeInterpreterAgent` | agent loop retry |
| `ExecutionSandbox` | supporting class | In-process executor. Runs approved Python code against the data payload; enforces wall-clock timeout and memory budget; returns `SandboxResult`. | invoked by `InterpretationWorkflow.executeStep` | returns result |
| `JobView` | `View` | Read model: one row per job for the UI. | `JobEntity` events | `JobEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record JobRequest(
    String jobId,
    String prompt,
    String dataPayload,        // raw user-supplied data (CSV / JSON / numeric text)
    DataFormat dataFormat,     // CSV | JSON | NUMERIC
    String submittedBy,
    Instant submittedAt
) {}
enum DataFormat { CSV, JSON, NUMERIC }

record GeneratedCode(
    String pythonSource,       // the code the agent produced
    List<String> importsList,  // modules the code imports; extracted at guard time
    Instant generatedAt
) {}

record SandboxResult(
    SandboxStatus status,      // COMPLETED | TIMED_OUT | MEMORY_EXCEEDED | RUNTIME_ERROR
    String answer,             // scalar answer string (may be empty for tabular output)
    List<List<String>> rows,   // tabular output (may be empty for scalar)
    List<String> columnNames,  // present when rows is non-empty
    String errorMessage,       // non-empty on failure
    long wallClockMillis
) {}
enum SandboxStatus { COMPLETED, TIMED_OUT, MEMORY_EXCEEDED, RUNTIME_ERROR }

record ExecutionResult(
    ResultKind kind,           // SCALAR | TABLE | ERROR
    String answer,
    List<List<String>> rows,
    List<String> columnNames,
    String pythonSourceRan,    // the approved code that actually ran
    Instant decidedAt
) {}
enum ResultKind { SCALAR, TABLE, ERROR }

record Job(
    String jobId,
    Optional<JobRequest> request,
    Optional<GeneratedCode> generatedCode,
    Optional<ExecutionResult> result,
    JobStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus {
    SUBMITTED, CODE_APPROVED, EXECUTING, RESULT_RECORDED, HALTED, FAILED
}
```

Events on `JobEntity`: `JobSubmitted`, `CodeApproved`, `ExecutionStarted`, `ResultRecorded`, `JobHalted`, `JobFailed`.

Every nullable lifecycle field on the `Job` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/jobs` — body `{ prompt, dataPayload, dataFormat, submittedBy }` → `{ jobId }`.
- `GET /api/jobs` — list all jobs, newest-first.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Code Interpreter Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted jobs (status pill + result-kind chip + age) and a right pane with the selected job's detail — the original prompt, data format chip, generated Python source, execution result (scalar answer or table), and wall-clock duration.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call the `CodeInterpreterAgent` emits before any code executes. Inspects the `pythonSource` for (1) forbidden imports (`os`, `sys`, `subprocess`, `socket`, `urllib`, `requests`, `importlib`, `ctypes`, `shutil`), (2) shell-escape calls (`os.system`, `subprocess.run`, `exec(`, `eval(` operating on strings from outer scope), (3) file-system writes outside the `/tmp/scratch/` prefix (`open(..., 'w')`, `open(..., 'a')`), and (4) direct network calls (`socket.connect`, `urllib.request`, `requests.get/post`). On any violation returns a structured rejection naming the check and the offending token; the agent loop retries within its iteration budget.
- **H1 — automatic safety halt**: enforced by `ExecutionSandbox` during `executeStep`. Every execution runs under a 10-second wall-clock budget and a 128 MB heap estimate. If the sandbox detects that either limit is exceeded, it terminates the execution thread, records `JobHalted` with the breach type, and transitions the entity to `HALTED`. The halt is final — no retry — and is visible in the UI as a red `HALTED` status pill.

## 9. Agent prompts

- `CodeInterpreterAgent` → `prompts/code-interpreter.md`. The single decision-making LLM. System prompt instructs it to generate a single, self-contained Python code block, import only allowed modules, and emit the code as a `code-execution` tool call argument rather than inline text.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a CSV prompt; within 30 s the result appears with a non-empty `answer` or a populated `rows` table, and the Python source that ran is shown.
2. **J2** — The agent's first code attempt imports `requests`; the `before-tool-call` guardrail rejects it; the second iteration produces code that imports only `csv` and `statistics`; the result lands.
3. **J3** — The agent generates a code block containing `while True: pass`; the sandbox halt fires within 10 s; the entity transitions to `HALTED`; the UI shows a red status pill and the halt reason.
4. **J4** — A job submitted with `dataFormat = JSON` produces a `ResultKind.TABLE` output with correct `columnNames` and row count matching the input array length.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named code-interpreter-agent demonstrating the single-agent × dev-code cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-code-interpreter-agent. Java package
io.akka.samples.agentwithcodeinterpretation. Akka 3.6.0. HTTP port 9673.

Components to wire (exactly):

- 1 AutonomousAgent CodeInterpreterAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/code-interpreter.md>) and
  .capability(TaskAcceptance.of(InterpretationTasks.INTERPRET_DATA).maxIterationsPerTask(3)).
  The task receives the user prompt as its instruction text and the data payload as a task
  ATTACHMENT (NOT as inline prompt text — TaskDef.attachment(name, contentBytes) is the
  canonical call). The agent emits a code-execution tool call; the workflow captures the
  generated code from the tool call arguments, runs the guardrail, then executes via
  ExecutionSandbox. Output: ExecutionResult{kind: ResultKind, answer: String,
  rows: List<List<String>>, columnNames: List<String>, pythonSourceRan: String,
  decidedAt: Instant}. The agent is configured with a before-tool-call guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries the response within its 3-iteration budget.

- 1 Workflow InterpretationWorkflow per jobId with two steps:
  * guardStep — calls componentClient.forAutonomousAgent(CodeInterpreterAgent.class,
    "interpreter-" + jobId).runSingleTask(
      TaskDef.instructions(job.request.prompt)
        .attachment("data-payload." + extension(job.request.dataFormat),
                    job.request.dataPayload.getBytes())
    ) — the before-tool-call guardrail is wired on the agent and fires inline before the
    tool call executes. On approval advances to executeStep with the approved GeneratedCode.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(InterpretationWorkflow::error).
  * executeStep — calls ExecutionSandbox.run(generatedCode.pythonSource,
    job.request.dataPayload, job.request.dataFormat) with a 10 s wall-clock budget and
    128 MB heap cap. On COMPLETED produces ExecutionResult and calls
    JobEntity.recordResult(result). On TIMED_OUT or MEMORY_EXCEEDED calls
    JobEntity.halt(breachType) — no retry. On RUNTIME_ERROR calls
    JobEntity.recordResult with kind=ERROR. WorkflowSettings.stepTimeout 15s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity JobEntity (one per jobId). State Job{jobId: String,
  request: Optional<JobRequest>, generatedCode: Optional<GeneratedCode>,
  result: Optional<ExecutionResult>, status: JobStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. JobStatus enum: SUBMITTED, CODE_APPROVED, EXECUTING,
  RESULT_RECORDED, HALTED, FAILED. Events: JobSubmitted{request}, CodeApproved{generatedCode},
  ExecutionStarted{}, ResultRecorded{result}, JobHalted{reason: String, breachType: String},
  JobFailed{reason}. Commands: submit, approveCode, markExecuting, recordResult, halt, fail,
  getJob. emptyState() returns Job.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View JobView with row type JobRow (mirrors Job minus request.dataPayload — the payload
  can be large; the view omits it and the UI fetches the full record on demand via
  GET /api/jobs/{id}). Table updater consumes JobEntity events. ONE query getAllJobs:
  SELECT * AS jobs FROM job_view. No WHERE status filter — Akka cannot auto-index enum
  columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * JobEndpoint at /api with POST /jobs (body {prompt, dataPayload, dataFormat, submittedBy};
    mints jobId; calls JobEntity.submit; starts InterpretationWorkflow; returns {jobId}),
    GET /jobs (list from getAllJobs, sorted newest-first), GET /jobs/{id} (one full job row),
    GET /jobs/sse (Server-Sent Events from the view's stream-updates), and three
    /api/metadata/* endpoints serving YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- InterpretationTasks.java declaring one Task<R> constant: INTERPRET_DATA = Task
  .name("Interpret data").description("Generate Python code to answer the prompt and return
  an ExecutionResult").resultConformsTo(ExecutionResult.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records JobRequest, DataFormat, GeneratedCode, SandboxResult, SandboxStatus,
  ExecutionResult, ResultKind, Job, JobStatus.

- CodeGuardrail.java implementing the before-tool-call hook. Extracts the pythonSource from
  the agent's tool call arguments. Checks (1) forbidden imports, (2) shell escapes,
  (3) out-of-scope file writes, (4) network calls. Returns Guardrail.reject(<structured-error>)
  naming the check and the offending token, or passes through. Wired via the agent's
  guardrail-configuration block.

- ExecutionSandbox.java — in-process Python simulation. Runs against the data payload with a
  10 s wall-clock timer (ScheduledExecutorService) and a synthetic 128 MB heap check
  (data payload size heuristic). For the generated scaffold, the executor interprets a
  safe subset of operations (count, sum, mean, max, min, sort, filter on numeric/CSV data)
  without spawning a real Python interpreter — this keeps the sample self-contained.
  Javadoc on the class documents how to replace the simulation with a real subprocess-isolated
  Python runtime for production use.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9673 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CodeInterpreterAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-datasets.jsonl with 3 seeded datasets:
  a 20-row CSV sales table, a JSON array of 15 API-latency metrics objects, and a
  numeric array of 50 response-time measurements. Each dataset is paired with 2–3
  example prompts that exercise the agent (aggregate, filter, sort).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false (user data
  payloads are not assumed to contain PII by default — the deployer must assess),
  decisions.authority_level = automated-execution (the code runs without human approval
  of individual runs), oversight.human_in_loop = false (the system is automated),
  failure.failure_modes including "malicious-code-generation", "runaway-execution",
  "forbidden-import-bypass", "incorrect-computation"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/code-interpreter.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Code Interpreter Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of job cards; right = selected-job detail with prompt, data format chip,
  generated Python source in a monospace block, execution result as scalar answer or table,
  and wall-clock duration chip). Browser title exactly:
  <title>Akka Sample: Code Interpreter Agent</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(jobId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    interpret-data.json — 8 ExecutionResult entries covering the three ResultKind values.
      Each entry has a non-empty pythonSource (valid Python — the sandbox runs it), an
      answer string for SCALAR entries, and a rows/columnNames pair for TABLE entries.
      Severities vary: some results are SCALAR sums, some are TABLE top-5 filters, some
      are ERROR (division by zero on empty dataset). Plus 2 deliberately BAD code entries
      (one importing `os`, one containing `socket.connect`) so the guardrail has work to
      do. The mock selects a bad entry on the FIRST iteration of every 3rd job (modulo
      seed) so J2 is reproducible.
- A MockModelProvider.seedFor(jobId) helper makes per-job selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeInterpreterAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion InterpretationTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (guardStep
  60s, executeStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the Job row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: InterpretationTasks.java with INTERPRET_DATA = Task.name(...).description(...)
  .resultConformsTo(ExecutionResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9673 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeInterpreterAgent).
  The ExecutionSandbox is a plain Java class — not an agent, not an LLM call — keeping
  the pattern's "one agent" promise honest.
- The data payload is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated guardStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns. It fires on the tool call
  the agent emits before any code executes.
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
