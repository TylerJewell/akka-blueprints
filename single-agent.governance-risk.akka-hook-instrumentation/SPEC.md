# SPEC — hook-instrumentation

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** HookInstrumentation.
**One-line pitch:** A user submits a task to an AI agent; every tool call and LLM invocation passes through a named hook chain — before-tool-call, after-tool-call, and before-llm-call — so that observers can block, sanitize, and log each activity without touching the agent's core decision logic.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. One `ActivityObserverAgent` (AutonomousAgent) executes the task; the surrounding hook chain enforces policy on every activity boundary. Three governance controls are wired around the agent:

- A **before-tool-call guardrail** runs before each tool invocation inside the agent's loop. It checks the requested tool name against an allowlist, validates input parameters against a schema, and blocks disallowed calls by returning a structured rejection that the agent receives as an error and must route around.
- An **after-tool-call guardrail** runs after each tool returns and before the result re-enters the agent's context. It scans the output for sensitive patterns (credentials, internal hostnames, PII-like tokens) and redacts them. The original output is written to the entity audit log; the agent only sees the sanitized form.
- A **before-llm-call guardrail** runs before each LLM invocation inside the agent loop. It inspects the assembled prompt for credential-like tokens and high-entropy strings that resemble secrets and strips them before the prompt leaves the process.

The blueprint shows that hooks are not bolted-on after the fact — they are first-class observers registered on the agent's definition that fire at deterministic activity boundaries. A well-defined hook chain lets a governance operator audit what the agent called and what it saw, without reading or modifying the agent's prompt.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **task template** from a dropdown (three seeded options: `summarize-report`, `calculate-metrics`, `lookup-policy`) or types a custom task description.
2. The user picks an **agent persona** (default `observer-standard`) and optionally ticks a **simulate blocked tool** checkbox to exercise the guardrail rejection path.
3. The user clicks **Submit task**. The UI POSTs to `/api/observations` and receives an `observationId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `HOOK_CHAIN_READY` — the hook configuration is visible in the card detail.
5. Within ~10–30 s the agent completes. The card transitions through `EXECUTING` → `COMPLETED`. The result appears: a top-level outcome badge (`SUCCESS` / `BLOCKED` / `PARTIAL`), the agent's final response text, and a hook-log table (one row per hook firing: hook name, activity type, verdict, redacted payload, timestamp).
6. Within ~1 s of completion, the observation scoring step finishes. The card shows a **coverage score** chip (1–5) plus a one-line note describing hook firing completeness.
7. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ObservationEndpoint` | `HttpEndpoint` | `/api/observations/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ObservationEntity`, `ObservationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ObservationEntity` | `EventSourcedEntity` | Per-task lifecycle: submitted → hook_chain_ready → executing → completed / failed. Source of truth including full hook log. | `ObservationEndpoint`, `HookLogConsumer`, `ObservationWorkflow` | `ObservationView` |
| `HookLogConsumer` | `Consumer` | Subscribes to `TaskSubmitted` events; initializes hook chain config; starts `ObservationWorkflow`. | `ObservationEntity` events | `ObservationEntity`, `ObservationWorkflow` |
| `ObservationWorkflow` | `Workflow` | One workflow per task. Steps: `hookInitStep` → `executeStep` → `coverageStep`. | started by `HookLogConsumer` | `ActivityObserverAgent`, `ObservationEntity` |
| `ActivityObserverAgent` | `AutonomousAgent` | The one decision-making LLM. Receives task definition with tool access; hook observers intercept every tool call and LLM invocation. | invoked by `ObservationWorkflow` | returns `AgentOutcome` |
| `ObservationView` | `View` | Read model: one row per observation for the UI. | `ObservationEntity` events | `ObservationEndpoint` |

Supporting classes (not Akka primitives):

| Class | Role |
|---|---|
| `BeforeToolCallGuardrail` | Validates tool name + input params; blocks disallowed calls. |
| `AfterToolCallGuardrail` | Redacts sensitive patterns from tool output before re-entry. |
| `BeforeLlmCallGuardrail` | Strips credential-like tokens from assembled prompts. |
| `CoverageScorer` | Deterministic rule-based scorer; checks hook log completeness. |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ToolSpec(String toolName, String description, List<String> allowedCallers) {}

record TaskRequest(
    String observationId,
    String taskDescription,
    String personaId,
    List<ToolSpec> availableTools,
    boolean simulateBlockedTool,
    String submittedBy,
    Instant submittedAt
) {}

record HookConfig(
    List<String> blockedToolNames,
    List<String> sensitivePatterns,   // regex patterns for after-tool-call redaction
    List<String> credentialPatterns   // regex patterns for before-llm-call scrubbing
) {}

record HookLogEntry(
    String entryId,
    HookPoint hookPoint,
    String toolNameOrPhase,
    HookVerdict verdict,
    String originalPayloadHash,   // SHA-256 of original; stored for audit, not the value
    String sanitizedPayload,      // what the agent actually saw
    String rejectReason,          // non-null when verdict == BLOCKED
    Instant firedAt
) {}
enum HookPoint { BEFORE_TOOL_CALL, AFTER_TOOL_CALL, BEFORE_LLM_CALL }
enum HookVerdict { ALLOWED, BLOCKED, REDACTED, PASS_THROUGH }

record AgentOutcome(
    Outcome outcome,
    String responseText,
    List<HookLogEntry> hookLog,
    Instant completedAt
) {}
enum Outcome { SUCCESS, BLOCKED, PARTIAL }

record CoverageScore(
    int score,            // 1..5
    String note,
    Instant scoredAt
) {}

record Observation(
    String observationId,
    Optional<TaskRequest> request,
    Optional<HookConfig> hookConfig,
    Optional<AgentOutcome> outcome,
    Optional<CoverageScore> coverage,
    ObservationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ObservationStatus {
    SUBMITTED, HOOK_CHAIN_READY, EXECUTING, COMPLETED, FAILED
}
```

Events on `ObservationEntity`: `TaskSubmitted`, `HookChainInitialized`, `ExecutionStarted`, `OutcomeRecorded`, `CoverageScored`, `ObservationFailed`.

Every nullable lifecycle field on the `Observation` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/observations` — body `{ taskDescription, personaId, availableTools: [ToolSpec], simulateBlockedTool, submittedBy }` → `{ observationId }`.
- `GET /api/observations` — list all observations, newest-first.
- `GET /api/observations/{id}` — one observation.
- `GET /api/observations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Hook Instrumentation</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted observations (status pill + outcome badge + age) and a right pane with the selected observation's detail — task description, hook configuration, agent outcome text, and a hook-log table.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`): runs before every tool invocation inside the `ActivityObserverAgent` loop. Checks the requested tool name against `HookConfig.blockedToolNames`; validates input parameters against the `ToolSpec` schema. Blocked calls return a structured rejection to the agent loop so the agent can recover or fail gracefully within its iteration budget.
- **G2 — after-tool-call guardrail** (`guardrail`, `after-tool-call`): runs after every tool returns, before the result re-enters the agent's context. Scans the output string for patterns in `HookConfig.sensitivePatterns`; redacts matches with a `[REDACTED-<PATTERN_NAME>]` token. Logs the original payload hash (SHA-256) to `HookLogEntry.originalPayloadHash` — the value never leaves the process; only the hash goes to the entity audit log.
- **G3 — before-llm-call guardrail** (`guardrail`, `before-llm-call`): runs before each LLM invocation inside the agent loop. Scans the assembled prompt for patterns in `HookConfig.credentialPatterns` (high-entropy strings, `Bearer` tokens, key-like sequences) and strips them. If stripping materially degrades the prompt, the guardrail records a `REDACTED` verdict; if the prompt is clean, it records `PASS_THROUGH`. Either way the scrubbed prompt is the only form the model sees.

## 9. Agent prompts

- `ActivityObserverAgent` → `prompts/activity-observer.md`. The single decision-making LLM. System prompt instructs it to execute the task using available tools, handle tool-call rejections gracefully (the guardrail will send back a structured error when a blocked tool is attempted), and return a final `AgentOutcome` with a response text and the accumulated hook log.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the `summarize-report` seed task; within 30 s the observation completes with `SUCCESS`, a hook-log table showing at least one `BEFORE_TOOL_CALL` + `AFTER_TOOL_CALL` entry, and a coverage score chip.
2. **J2** — User submits with `simulateBlockedTool = true`; the `before-tool-call` guardrail fires a `BLOCKED` verdict for the disallowed tool; the agent recovers; the hook log shows the `BLOCKED` entry; the outcome is `PARTIAL` or `SUCCESS` (not `FAILED`).
3. **J3** — A tool returns output containing a string matching a `sensitivePatterns` entry; the agent's conversation log shows only the `[REDACTED-*]` form; the entity stores only the hash of the original.
4. **J4** — The assembled LLM prompt for a step contains a `Bearer eyJ...` token; the `before-llm-call` guardrail strips it; the model call log shows `[REDACTED-CREDENTIAL]` in that position.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hook-instrumentation demonstrating the single-agent × governance-risk cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-governance-risk-akka-hook-instrumentation. Java package
io.akka.samples.hooksprepostactivity. Akka 3.6.0. HTTP port 9891.

Components to wire (exactly):

- 1 AutonomousAgent ActivityObserverAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/activity-observer.md>) and
  .capability(TaskAcceptance.of(EXECUTE_TASK).maxIterationsPerTask(5)).
  Three hook observers registered on the agent definition:
    * BeforeToolCallGuardrail bound to the before-tool-call hook. Reads HookConfig.blockedToolNames
      from the task context; if the requested tool is in that list returns Guardrail.reject(
      structured error message naming the blocked tool). Otherwise validates the tool input
      parameters against the matching ToolSpec; on schema mismatch returns a structured rejection.
      Passing calls are logged as HookLogEntry(BEFORE_TOOL_CALL, ALLOWED, ...).
    * AfterToolCallGuardrail bound to the after-tool-call hook. Scans the tool output string against
      HookConfig.sensitivePatterns; for each match replaces the matched span with
      [REDACTED-<PATTERN_NAME>] and logs a HookLogEntry(AFTER_TOOL_CALL, REDACTED, ...) storing
      SHA-256(originalOutput) as originalPayloadHash. Clean output logs as HookLogEntry(
      AFTER_TOOL_CALL, PASS_THROUGH, ...).
    * BeforeLlmCallGuardrail bound to the before-llm-call hook. Scans the assembled prompt string
      against HookConfig.credentialPatterns; strips matched spans; logs HookLogEntry(
      BEFORE_LLM_CALL, REDACTED or PASS_THROUGH, ...). If stripping removes more than 10% of
      the prompt token count, records a REDACTED verdict and a note in rejectReason.
  The guardrails accumulate HookLogEntry records into the task's shared context; the
  agent's final response carries the full hookLog list inside AgentOutcome.
  Output type: AgentOutcome{outcome: Outcome (SUCCESS/BLOCKED/PARTIAL), responseText: String,
  hookLog: List<HookLogEntry>, completedAt: Instant}.

- 1 Workflow ObservationWorkflow per observationId with three steps:
  * hookInitStep — reads ObservationEntity.getObservation; when request.isPresent() builds the
    HookConfig from the task's availableTools and a seeded blockedToolNames list, emits
    HookChainInitialized, advances to executeStep. WorkflowSettings.stepTimeout 10s.
  * executeStep — emits ExecutionStarted, then calls componentClient.forAutonomousAgent(
    ActivityObserverAgent.class, "observer-" + observationId).runSingleTask(
      TaskDef.instructions(formatTask(request.taskDescription, request.availableTools))
    ) — returns a taskId, then forTask(taskId).result(EXECUTE_TASK) to fetch the outcome.
    On success calls ObservationEntity.recordOutcome(outcome). WorkflowSettings.stepTimeout 90s
    with defaultStepRecovery maxRetries(2).failoverTo(ObservationWorkflow::error).
  * coverageStep — runs CoverageScorer (deterministic, no LLM) over the recorded AgentOutcome:
    checks that hook entries exist for every tool call (before + after pair), that no
    BEFORE_LLM_CALL entry is missing for each LLM invocation, and that BLOCKED entries include
    non-empty rejectReason. Score 1–5; note one sentence. Emits CoverageScored. WorkflowSettings
    .stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ObservationEntity (one per observationId). State Observation{observationId:
  String, request: Optional<TaskRequest>, hookConfig: Optional<HookConfig>, outcome:
  Optional<AgentOutcome>, coverage: Optional<CoverageScore>, status: ObservationStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. ObservationStatus enum: SUBMITTED,
  HOOK_CHAIN_READY, EXECUTING, COMPLETED, FAILED. Events: TaskSubmitted{request},
  HookChainInitialized{hookConfig}, ExecutionStarted{}, OutcomeRecorded{outcome},
  CoverageScored{coverage}, ObservationFailed{reason}. Commands: submit, initHookChain,
  markExecuting, recordOutcome, recordCoverage, fail, getObservation. emptyState() returns
  Observation.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer HookLogConsumer subscribed to ObservationEntity events; on TaskSubmitted
  calls ObservationEntity.initHookChain(buildHookConfig(request)) and then starts an
  ObservationWorkflow with id = "obs-" + observationId.

- 1 View ObservationView with row type ObservationRow (mirrors Observation minus
  request details — full detail on-demand via GET /api/observations/{id}). Table updater
  consumes ObservationEntity events. ONE query getAllObservations: SELECT * AS observations
  FROM observation_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ObservationEndpoint at /api with POST /observations (body
    {taskDescription, personaId, availableTools: [{toolName, description, allowedCallers}],
    simulateBlockedTool, submittedBy}; mints observationId; calls ObservationEntity.submit;
    returns {observationId}), GET /observations (list from getAllObservations, sorted
    newest-first), GET /observations/{id} (one row), GET /observations/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ObservationTasks.java declaring one Task<R> constant: EXECUTE_TASK = Task.name("Execute task")
  .description("Execute the described task using the available tools; log every hook firing into
  the hookLog; return an AgentOutcome").resultConformsTo(AgentOutcome.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ToolSpec, TaskRequest, HookConfig, HookLogEntry, HookPoint, HookVerdict,
  AgentOutcome, Outcome, CoverageScore, Observation, ObservationStatus.

- BeforeToolCallGuardrail.java, AfterToolCallGuardrail.java, BeforeLlmCallGuardrail.java —
  each implementing the matching hook interface. See eval-matrix.yaml G1, G2, G3 for the exact
  check logic.

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: AgentOutcome, TaskRequest.
  Outputs: CoverageScore. Scoring rubric documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9891 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ActivityObserverAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/tool-registry.jsonl with 3 seeded tool sets:
  a 3-tool summarizer set (read-file, extract-key-points, format-output),
  a 3-tool calculator set (parse-expression, compute-result, format-number),
  and a 3-tool policy-lookup set (search-policy-index, fetch-policy-text, summarize-policy).
  Each set includes one tool flagged as blocked in the seeded HookConfig.

- src/main/resources/sample-events/seed-tasks.jsonl with 3 paired example task descriptions
  and tool sets matching the templates above.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, G2, G3) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes pre-filled per domain.

- prompts/activity-observer.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Hook Instrumentation", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of observation cards; right = selected-observation detail with task description,
  hook config summary, outcome text, and hook-log table).
  Browser title exactly: <title>Akka Sample: Hook Instrumentation</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on EXECUTE_TASK.
- Per-task mock-response shapes:
    execute-task.json — 6 AgentOutcome entries covering all three Outcome values.
    Each entry has a responseText paragraph and a hookLog with realistic entries per tool in
    the matching tool set. At least 2 entries have a BLOCKED hook log entry (the blocked tool
    was attempted). Plus 1 entry where BEFORE_LLM_CALL fires REDACTED (a credential token was
    present in the assembled prompt). The mock selects a BLOCKED entry on the FIRST iteration
    of every 3rd submission so J2 is reproducible.
- MockModelProvider.seedFor(observationId) makes per-task selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ActivityObserverAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ObservationTasks.java MUST exist.
- Lesson 4: every workflow step has explicit stepTimeout (hookInitStep 10s, executeStep 90s,
  coverageStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Observation row record is Optional<T>.
- Lesson 7: ObservationTasks.java with EXECUTE_TASK = Task.name(...).description(...)
  .resultConformsTo(AgentOutcome.class) is mandatory.
- Lesson 8: model names — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9891 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute only. Exactly five tab-panel
  sections.
- The single-agent invariant: exactly ONE AutonomousAgent (ActivityObserverAgent). The
  coverage scorer is rule-based (CoverageScorer.java) and does NOT make an LLM call.
- All three guardrails are registered on the agent's definition, not as external post-return
  checks.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
