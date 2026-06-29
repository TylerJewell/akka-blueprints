# SPEC — self-improving-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Improving Agent.
**One-line pitch:** Submit a task; an executor agent runs it and records performance signals; an optimizer agent analyzes those signals and proposes a prompt revision; the workflow attests and applies the revision or halts when the ceiling is reached.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between an executor agent (`ExecutorAgent`) that processes tasks under the current configuration and an optimizer agent (`OptimizerAgent`) that analyzes the resulting performance records and proposes revisions. A build-gate attestation control blocks any configuration change until a regression test against a held-out task set passes. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, a **ci-gate** that requires a passing attestation before a revision is applied, and a **hotl** (human-on-the-loop) monitor that receives a notification on every applied or rejected revision so a human operator can review the agent's self-modification history.

## 3. User-facing flows

The user opens the App UI tab and submits a task (a short free-text instruction plus an expected-output hint).

1. The system creates a `TaskBatch` record in `COLLECTING` and accumulates results until a configurable batch size is reached.
2. `ExecutorAgent` processes each task using the current `AgentConfig` prompt and returns an `ExecutionResult` with `outputText`, `qualityScore` (1–5), and `latencyMs`.
3. Once the batch is complete, `ImprovementWorkflow` starts. It reads the current `AgentConfig` and calls `OptimizerAgent` with the batch's `PerformanceRecord` list.
4. `OptimizerAgent` proposes a `PromptRevision` — a revised system-prompt fragment and a one-paragraph rationale.
5. The **attestation gate** runs a regression check: `ExecutorAgent` re-runs a held-out reference task set under the proposed config. If the mean quality score improves by at least the configured threshold (default 0.5), the gate emits `AttestationPassed`.
6. On `AttestationPassed`, the workflow emits `RevisionApplied`, updates `AgentConfigEntity` to the new config, and transitions the revision to `APPLIED`.
7. On gate failure, the workflow records `AttestationFailed` with the delta score, increments the revision attempt counter, and calls `OptimizerAgent` again with the failed attestation result attached. If the loop reaches `maxRevisions` (default 3) without a passing gate, the workflow emits `RevisionRejected` with the best-scoring proposed revision preserved for audit.

A `TaskSimulator` (TimedAction) drips a canned task every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ExecutorAgent` | `AutonomousAgent` | Processes a task under the current config; returns `ExecutionResult`. | `ImprovementWorkflow` | returns `ExecutionResult` to workflow |
| `OptimizerAgent` | `AutonomousAgent` | Analyzes a `PerformanceRecord` batch; proposes a `PromptRevision`. | `ImprovementWorkflow` | returns `PromptRevision` to workflow |
| `ImprovementWorkflow` | `Workflow` | Runs the evaluate → optimize → attest → apply loop; halts at the ceiling. | `AgentEndpoint`, `TaskResultConsumer` | `AgentConfigEntity` |
| `AgentConfigEntity` | `EventSourcedEntity` | Holds the current config, every proposal, every attestation, and the revision lineage. | `ImprovementWorkflow` | `ConfigView` |
| `TaskQueueEntity` | `EventSourcedEntity` | Logs each submitted task for replay and audit. | `AgentEndpoint`, `TaskSimulator` | `TaskResultConsumer` |
| `ConfigView` | `View` | Read model of active configs and revision history. | `AgentConfigEntity` events | `AgentEndpoint` |
| `TaskResultConsumer` | `Consumer` | Subscribes to `TaskQueueEntity` events; starts a workflow when a batch is ready. | `TaskQueueEntity` events | `ImprovementWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task every 60 s from `sample-events/tasks.jsonl`. | scheduler | `TaskQueueEntity` |
| `AttestationSampler` | `TimedAction` | Every 30 s, scans `ConfigView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `AgentConfigEntity` |
| `AgentEndpoint` | `HttpEndpoint` | `/api/configs/*` — submit task, list configs, get config, SSE; plus `/api/metadata/*`. | — | `ConfigView`, `TaskQueueEntity`, `AgentConfigEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Task(String taskId, String instruction, String expectedHint, String submittedBy) {}

record ExecutionResult(String outputText, int qualityScore, long latencyMs, Instant executedAt) {}

record PerformanceRecord(
    String configId,
    List<ExecutionResult> results,
    double meanQualityScore,
    Instant collectedAt
) {}

record PromptRevision(
    String proposedSystemPrompt,
    String rationale,
    int revisionAttempt
) {}

record AttestationVerdict(boolean passed, double scoreDelta, String detail) {}

record RevisionCycle(
    int cycleNumber,
    PromptRevision proposal,
    AttestationVerdict attestation,
    Optional<String> appliedConfigId
) {}

record AgentConfig(
    String configId,
    String systemPrompt,
    AgentConfigStatus status,
    List<RevisionCycle> cycles,
    Optional<String> parentConfigId,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finalizedAt
) {}

enum AgentConfigStatus { ACTIVE, IMPROVING, APPLIED, REVISION_REJECTED }

enum AttestationResult { PASSED, FAILED }
```

### Events (on `AgentConfigEntity`)

`ConfigInitialized`, `RevisionProposed`, `AttestationRecorded`, `RevisionApplied`, `RevisionRejected`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ instruction, expectedHint?, submittedBy? }` → `{ taskId }`. Enqueues a task.
- `GET /api/configs` — list all agent configurations. Optional `?status=ACTIVE|IMPROVING|APPLIED|REVISION_REJECTED`.
- `GET /api/configs/{id}` — one configuration (including every revision cycle, proposal, and attestation verdict).
- `GET /api/configs/sse` — server-sent events stream of every config change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Self-Improving Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, ci-gate = red, hotl = orange).
- **App UI** — form to submit a task, live list of configs with status pills, click-to-expand per-cycle timeline showing each proposal, the attestation verdict, and the applied status.

Browser title: `<title>Akka Sample: Self-Improving Agent</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every cycle's attestation verdict is recorded as an `EvalRecorded` event with `{ cycleNumber, attestationResult, scoreDelta, applied }`. The `AttestationSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking.
- **G1 — ci-gate** (`attestation-gate`): before any proposed `PromptRevision` is applied, the workflow re-runs the `ExecutorAgent` against a held-out reference task set under the proposed config. The mean quality score must improve by at least `self-improving-agent.improvement.threshold` (default 0.5) for `AttestationPassed` to emit. If the gate fails, the proposed revision is discarded; the current config remains active. Enforcement: build-gate.
- **HM1 — hotl** (`deployer-runtime-monitoring`): every `RevisionApplied` or `RevisionRejected` event triggers a notification event that a human-on-the-loop monitor can subscribe to. The `AgentEndpoint` exposes `/api/configs/monitor/sse` as a dedicated stream for monitoring tooling. Enforcement: system-level.

## 9. Agent prompts

- `ExecutorAgent` → `prompts/executor.md`. Processes a task under the current system-prompt config; returns `ExecutionResult` with output text and self-scored quality.
- `OptimizerAgent` → `prompts/optimizer.md`. Analyzes a batch of `PerformanceRecord` entries; proposes a `PromptRevision` with a revised system-prompt fragment and a one-paragraph rationale.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit tasks; after batch completion the optimizer proposes a revision; attestation passes on or before `maxRevisions`; the App UI shows the new config applied and every cycle's proposal and attestation.
2. **J2 — halt at ceiling** — Force the attestation gate to always fail (use the `"test-force-reject"` instruction prefix, which the mock provider's seedFor logic always answers with a failing verdict); the workflow cycles `maxRevisions` times and lands in `REVISION_REJECTED` with the best-scoring proposed config preserved.
3. **J3 — eval-event timeline** — The expanded config view shows one `EvalRecorded` event per completed cycle with `attestationResult`, `scoreDelta`, and `applied` populated.
4. **J4 — monitor stream** — A client connected to `/api/configs/monitor/sse` receives a notification event within 5 s of any `RevisionApplied` or `RevisionRejected` transition.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-improving-agent demonstrating the evaluator-optimizer ×
general domain cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-self-improving-agent.
Java package io.akka.samples.selfimprovingagents. Akka 3.6.0. HTTP port 9223.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ExecutorAgent — definition() with
    capability(TaskAcceptance.of(EXECUTE_TASK).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REGRESSION_EXECUTE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/executor.md. Returns ExecutionResult{outputText,
    qualityScore, latencyMs, executedAt} for both EXECUTE_TASK and REGRESSION_EXECUTE.
    The REGRESSION_EXECUTE task takes (referenceInstruction, proposedSystemPrompt) as
    additional inputs.
  * OptimizerAgent — definition() with
    capability(TaskAcceptance.of(PROPOSE_REVISION).maxIterationsPerTask(2)). System
    prompt from prompts/optimizer.md. Returns PromptRevision{proposedSystemPrompt,
    rationale, revisionAttempt} where revisionAttempt is the 1-indexed count.

- 1 Workflow ImprovementWorkflow with steps:
    collectStep -> executeStep -> evaluateStep -> proposeStep -> attestStep ->
    [gate PASSED? applyStep : (revisionAttempt < maxRevisions ?
       proposeStep with failed attestation attached : rejectStep)] -> END.
  executeStep calls forAutonomousAgent(ExecutorAgent.class, configId).runSingleTask(
    EXECUTE_TASK) then forTask(taskId).result(EXECUTE_TASK). proposeStep calls
    forAutonomousAgent(OptimizerAgent.class, configId).runSingleTask(PROPOSE_REVISION).
  attestStep calls forAutonomousAgent(ExecutorAgent.class, configId).runSingleTask(
    REGRESSION_EXECUTE) for each reference task (up to 3), computes mean quality
    score delta vs. current config baseline, emits AttestationRecorded with
    AttestationVerdict{passed, scoreDelta, detail}. applyStep emits RevisionApplied
    and updates AgentConfigEntity to the new config. rejectStep emits RevisionRejected
    with the highest-scoring proposed revision's text as best-of and a structured
    rejectionReason. Override settings() with stepTimeout(90s) on executeStep,
    proposeStep, and attestStep, and defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).

- 1 EventSourcedEntity AgentConfigEntity holding state AgentConfig{configId,
  systemPrompt, AgentConfigStatus status, List<RevisionCycle> cycles,
  Optional<String> parentConfigId, Optional<String> rejectionReason,
  Instant createdAt, Optional<Instant> finalizedAt}.
  AgentConfigStatus enum: ACTIVE, IMPROVING, APPLIED, REVISION_REJECTED.
  Events: ConfigInitialized, RevisionProposed, AttestationRecorded,
  RevisionApplied, RevisionRejected, EvalRecorded.
  Commands: initConfig, proposeRevision, recordAttestation, applyRevision,
  rejectRevision, recordEval, getConfig. emptyState() returns AgentConfig.initial()
  with no commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity TaskQueueEntity with command submitTask(instruction,
  expectedHint, submittedBy) emitting TaskSubmitted{taskId, instruction,
  expectedHint, submittedBy, submittedAt}.

- 1 View ConfigView with row type ConfigRow (mirrors AgentConfig; the cycles list
  is preserved as-is — bounded at maxRevisions so size stays reasonable).
  Table updater consumes AgentConfigEntity events. ONE query
  getAllConfigs SELECT * AS configs FROM config_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer TaskResultConsumer subscribed to TaskQueueEntity events; on
  TaskSubmitted accumulates results until batch size reached, then starts
  an ImprovementWorkflow with the configId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/tasks.jsonl and calls
    TaskQueueEntity.submitTask.
  * AttestationSampler — every 30s, queries ConfigView.getAllConfigs, finds
    configs with a completed cycle that has not yet been recorded as an
    EvalRecorded event, and calls AgentConfigEntity.recordEval(cycleNumber,
    attestationResult, scoreDelta, applied). Idempotent per (configId, cycleNumber).

- 2 HttpEndpoints:
  * AgentEndpoint at /api with POST /tasks, GET /configs, GET /configs/{id},
    GET /configs/sse, GET /configs/monitor/sse, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/. The POST /tasks body
    is {instruction, expectedHint?, submittedBy?}; missing expectedHint defaults to "",
    missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- AgentTasks.java declaring three Task<R> constants: EXECUTE_TASK (resultConformsTo
  ExecutionResult), REGRESSION_EXECUTE (ExecutionResult), PROPOSE_REVISION (PromptRevision).
- Domain records ExecutionResult, PerformanceRecord, PromptRevision, AttestationVerdict,
  RevisionCycle, AgentConfig; enums AgentConfigStatus, AttestationResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9223
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from the
  canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  self-improving-agent.improvement.max-revisions = 3 and
  self-improving-agent.improvement.threshold = 0.5, overridable by env var.
- src/main/resources/sample-events/tasks.jsonl with 8 canned task lines, each shaped
  {"instruction":"...", "expectedHint":"..."}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (E1 eval-event on-decision-eval,
  G1 ci-gate attestation-gate, HM1 hotl deployer-runtime-monitoring) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = agent-self-optimization,
  decisions.authority_level = config-change-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/executor.md, prompts/optimizer.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Self-Improving Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-cycle timeline). Browser title exactly:
  <title>Akka Sample: Self-Improving Agent</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
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
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: executor.json, optimizer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    executor.json — 6 ExecutionResult entries. Three have qualityScore=4
      or 5 (high-quality outputs on in-domain tasks). Two have
      qualityScore=2 or 3 (outputs that expose prompt weaknesses — used
      to trigger the optimizer). One has qualityScore=1 (regression
      failure used to exercise the attestation gate's reject path in J2).
    optimizer.json — 6 PromptRevision entries. Three propose a
      revised system-prompt fragment that tightens the output format
      (shorter, more structured) with a one-paragraph rationale. Three
      propose a revision that widens the format (more verbose) and include
      a CritiqueNotes-style rationale noting "previous outputs lacked
      context in lines 2–3".
- A MockModelProvider.seedFor(configId, cycleNumber) helper makes the
  selection deterministic per (configId, cycleNumber) so the same config
  in dev produces the same improvement trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. ExecutorAgent
  and OptimizerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an AgentTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the AgentConfig row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: AgentTasks.java is mandatory; generating ExecutorAgent or
  OptimizerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9223, declared in application.conf dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
