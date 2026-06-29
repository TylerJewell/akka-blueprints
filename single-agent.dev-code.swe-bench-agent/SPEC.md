# SPEC — swe-bench-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SWE-Bench Agent.
**One-line pitch:** A user submits a bug description and a repository snapshot; one AI agent reads the snapshot (passed as a task attachment, never as inline prompt text) and produces a structured patch result — PASS / FAIL / NEEDS_RETRY with a diff, a per-test breakdown, and a confidence score.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `PatchEngineerAgent` (AutonomousAgent) carries the entire decision; the surrounding components only prepare its input and validate its output. One governance mechanism is wired around the agent:

- A **CI gate** runs the repository's test suite against every candidate patch before the result is accepted. If any test fails, the gate blocks the result and the workflow triggers a retry inside the same task iteration budget. The gate is deterministic — same patch, same tests, same result — which is what makes it a trustworthy check on LLM-generated code.

The blueprint shows that governance in a code-generation setting is about executable verification: a guardrail on output structure ensures the patch is parseable, and a test gate ensures it actually works.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks one of three seeded tasks from the **Task** dropdown (a Python `AttributeError` fix, a Go nil-pointer dereference fix, a TypeScript type-narrowing fix) or pastes a custom bug description.
2. The user picks a **Repository snapshot** (pre-bundled tarballs for each seeded task) or uploads a custom snapshot.
3. The user clicks **Submit task**. The UI POSTs to `/api/tasks` and receives a `taskId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SNAPSHOT_PREPARED` — the normalized snapshot is ready and the file-tree summary is visible in the card detail.
5. Within ~15–60 s, the workflow's `patchStep` completes. The card transitions to `PATCHING` then `PATCH_PRODUCED`. The diff appears in the card detail alongside a confidence score (0–100) the agent assigns.
6. Within ~5–10 s, the `testGateStep` runs the simulated test suite against the diff. The card transitions to `GATE_PASSED` or `GATE_FAILED`. The test report appears: pass/fail per test, total runtime, and a summary verdict.
7. On `GATE_PASSED`, the card reaches `COMPLETED`. On `GATE_FAILED`, the workflow retries the patch step (up to the retry budget) before reaching `FAILED`.
8. The user can submit another task; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BenchmarkEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BenchmarkTaskEntity`, `BenchmarkView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BenchmarkTaskEntity` | `EventSourcedEntity` | Per-task lifecycle: submitted → snapshot-prepared → patching → patch-produced → gate-passed/failed → completed/failed. Source of truth. | `BenchmarkEndpoint`, `SnapshotPreparer`, `BenchmarkWorkflow` | `BenchmarkView` |
| `SnapshotPreparer` | `Consumer` | Subscribes to `TaskSubmitted` events; normalizes the repository snapshot; calls `BenchmarkTaskEntity.attachPreparedSnapshot`. | `BenchmarkTaskEntity` events | `BenchmarkTaskEntity` |
| `BenchmarkWorkflow` | `Workflow` | One workflow per task. Steps: `awaitSnapshotStep` → `patchStep` → `testGateStep`. | started by `SnapshotPreparer` once prepared event lands | `PatchEngineerAgent`, `BenchmarkTaskEntity` |
| `PatchEngineerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the bug description in the task definition and the normalized snapshot as a task attachment; returns `PatchResult`. | invoked by `BenchmarkWorkflow` | returns patch result |
| `PatchGuardrail` | supporting class | Validates the agent's candidate `PatchResult` on each turn: well-formed diff block, confidence score in 0–100, file paths within the snapshot root. On failure rejects and forces retry. | invoked by `PatchEngineerAgent` loop | `PatchEngineerAgent` |
| `TestGateRunner` | supporting class | Rule-based simulated test runner. Parses the diff, applies it to the prepared snapshot, runs the bundled test stubs, produces a `GateReport`. No LLM call. | invoked by `BenchmarkWorkflow.testGateStep` | `BenchmarkWorkflow` |
| `BenchmarkView` | `View` | Read model: one row per task for the UI. | `BenchmarkTaskEntity` events | `BenchmarkEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BugDescription(
    String taskId,
    String title,
    String description,
    String language,           // "python" | "go" | "typescript"
    String repositoryName
) {}

record PreparedSnapshot(
    String snapshotRef,        // content-hash of the normalized tarball
    List<String> filePaths,    // relative paths in the snapshot root
    List<String> testFilePaths // relative paths of test files
) {}

record HunkChange(
    String filePath,
    int startLine,
    int endLine,
    String oldContent,
    String newContent
) {}

record PatchResult(
    String unifiedDiff,
    List<HunkChange> hunks,
    int confidenceScore,       // 0..100
    String patchSummary,
    Instant patchedAt
) {}

record TestCase(
    String testId,
    String testFile,
    TestOutcome outcome,
    String failureMessage      // null when outcome == PASS
) {}
enum TestOutcome { PASS, FAIL, ERROR }

record GateReport(
    GateVerdict verdict,
    List<TestCase> testCases,
    int totalTests,
    int passedTests,
    long runtimeMs,
    Instant completedAt
) {}
enum GateVerdict { PASS, FAIL }

record BenchmarkTask(
    String taskId,
    Optional<BugDescription> description,
    Optional<PreparedSnapshot> snapshot,
    Optional<PatchResult> patch,
    Optional<GateReport> gateReport,
    TaskStatus status,
    int attemptNumber,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus {
    SUBMITTED, SNAPSHOT_PREPARED, PATCHING, PATCH_PRODUCED,
    GATE_PASSED, GATE_FAILED, COMPLETED, FAILED
}
```

Events on `BenchmarkTaskEntity`: `TaskSubmitted`, `SnapshotPrepared`, `PatchingStarted`, `PatchProduced`, `GateReportRecorded`, `TaskCompleted`, `TaskFailed`.

Every nullable lifecycle field on the `BenchmarkTask` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/tasks` — body `{ title, description, language, repositoryName, snapshotBytes (base64) }` → `{ taskId }`.
- `GET /api/tasks` — list all tasks, newest-first.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SWE-Bench Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tasks (status pill + gate verdict badge + age) and a right pane with the selected task's detail — bug description, prepared snapshot file tree, patch diff, test case table, and gate verdict chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **C1 — CI gate** (`ci-gate`, applied inside `TestGateRunner` via `BenchmarkWorkflow.testGateStep`): runs the repository's test suite against every candidate patch before the result is accepted. A deterministic rule-based runner (no LLM call) parses the diff, applies it to the prepared snapshot, and executes the bundled test stubs. If any test fails, the gate returns `GateVerdict.FAIL`, the workflow retries the patch step (up to `maxRetries(2)`), and only records `COMPLETED` when a patch passes all tests.

## 9. Agent prompts

- `PatchEngineerAgent` → `prompts/patch-engineer.md`. The single decision-making LLM. System prompt instructs it to read the attached snapshot, understand the bug description, produce a unified diff that fixes the bug, and assign a confidence score.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Python AttributeError seed task; within 60 s the patch appears, the test gate passes, and the card reaches `COMPLETED`.
2. **J2** — The agent's first patch iteration fails the test gate (mock path) — the gate rejects it; the second iteration produces a passing patch; the UI shows `COMPLETED` and the gate report lists all tests as PASS.
3. **J3** — A patch whose diff touches files outside the snapshot root is rejected by `PatchGuardrail` before leaving the agent loop; the card retries within its iteration budget.
4. **J4** — A snapshot uploaded with injected API-key-like tokens is normalized before the LLM call; the LLM call log shows `[REDACTED-SECRET]` placeholders; the entity retains the raw snapshot ref for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named swe-bench-agent demonstrating the single-agent × dev-code cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-swe-bench-agent. Java package io.akka.samples.swebenchmarkagent.
Akka 3.6.0. HTTP port 9219.

Components to wire (exactly):

- 1 AutonomousAgent PatchEngineerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/patch-engineer.md>) and
  .capability(TaskAcceptance.of(PATCH_ISSUE).maxIterationsPerTask(3)). The task receives
  the bug description as its instruction text and the normalized snapshot as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes)
  is the canonical call). Output: PatchResult{unifiedDiff: String, hunks: List<HunkChange>,
  confidenceScore: int (0-100), patchSummary: String, patchedAt: Instant}. The agent is
  configured with a before-agent-response guardrail (see C1 in eval-matrix.yaml — note:
  the guardrail is separate from the CI gate; it checks structural validity of the patch
  output) registered via the agent's guardrail-configuration block. On guardrail rejection
  the agent loop retries within its 3-iteration budget.

- 1 Workflow BenchmarkWorkflow per taskId with three steps:
  * awaitSnapshotStep — polls BenchmarkTaskEntity.getTask every 1s; on
    task.snapshot().isPresent() advances to patchStep.
    WorkflowSettings.stepTimeout 15s.
  * patchStep — emits PatchingStarted, then calls componentClient.forAutonomousAgent(
    PatchEngineerAgent.class, "engineer-" + taskId).runSingleTask(
      TaskDef.instructions(formatBugDescription(task.description))
        .attachment("snapshot.tar.gz", task.snapshot.snapshotRef.getBytes())
    ) — returns a taskId, then forTask(taskId).result(PATCH_ISSUE) to fetch the patch.
    On success calls BenchmarkTaskEntity.recordPatch(patch). WorkflowSettings.stepTimeout
    120s with defaultStepRecovery maxRetries(2).failoverTo(BenchmarkWorkflow::error).
  * testGateStep — runs a deterministic rule-based TestGateRunner (NOT an LLM call)
    over the recorded patch: parses the unified diff, applies hunks to the prepared
    snapshot files, executes the bundled test stubs (in-process Java assertions that
    check expected output strings). Emits GateReportRecorded{report}. On GateVerdict.PASS
    emits TaskCompleted. On GateVerdict.FAIL retries patchStep (within the workflow
    retry budget) before transitioning to FAILED. WorkflowSettings.stepTimeout 30s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BenchmarkTaskEntity (one per taskId). State BenchmarkTask{taskId:
  String, description: Optional<BugDescription>, snapshot: Optional<PreparedSnapshot>,
  patch: Optional<PatchResult>, gateReport: Optional<GateReport>, status: TaskStatus,
  attemptNumber: int, createdAt: Instant, finishedAt: Optional<Instant>}.
  TaskStatus enum: SUBMITTED, SNAPSHOT_PREPARED, PATCHING, PATCH_PRODUCED, GATE_PASSED,
  GATE_FAILED, COMPLETED, FAILED.
  Events: TaskSubmitted{description}, SnapshotPrepared{snapshot}, PatchingStarted{},
  PatchProduced{patch}, GateReportRecorded{report}, TaskCompleted{}, TaskFailed{reason}.
  Commands: submit, attachPreparedSnapshot, markPatching, recordPatch, recordGateReport,
  complete, fail, getTask. emptyState() returns BenchmarkTask.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer SnapshotPreparer subscribed to BenchmarkTaskEntity events; on TaskSubmitted
  runs a normalization pipeline (strips binary files, sanitizes secret-like tokens
  matching common patterns — API keys, bearer tokens, private-key headers, connection
  strings — into [REDACTED-SECRET] placeholders, validates the file-tree depth) over
  the raw snapshot bytes, builds PreparedSnapshot (snapshotRef = SHA-256 hex of
  normalized content, filePaths, testFilePaths), then calls
  BenchmarkTaskEntity.attachPreparedSnapshot(snapshot). After that call lands, the same
  Consumer starts a BenchmarkWorkflow with id = "bench-" + taskId.

- 1 View BenchmarkView with row type BenchmarkRow (mirrors BenchmarkTask minus the raw
  snapshot bytes — the audit log keeps those; the view holds the snapshotRef and file
  lists for the UI). Table updater consumes BenchmarkTaskEntity events. ONE query
  getAllTasks: SELECT * AS tasks FROM benchmark_view. No WHERE status filter — Akka
  cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * BenchmarkEndpoint at /api with POST /tasks (body
    {title, description, language, repositoryName, snapshotBytes (base64)}; mints
    taskId; calls BenchmarkTaskEntity.submit; returns {taskId}), GET /tasks (list from
    getAllTasks, sorted newest-first), GET /tasks/{id} (one row), GET /tasks/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- BenchmarkTasks.java declaring one Task<R> constant: PATCH_ISSUE = Task.name("Patch
  issue").description("Read the attached repository snapshot and produce a PatchResult
  fixing the described bug").resultConformsTo(PatchResult.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records BugDescription, PreparedSnapshot, HunkChange, PatchResult, TestCase,
  TestOutcome, GateReport, GateVerdict, BenchmarkTask, TaskStatus.

- PatchGuardrail.java implementing the before-agent-response hook. Reads the candidate
  PatchResult from the LLM response, asserts (1) the unifiedDiff block is non-empty and
  begins with "---", (2) confidenceScore is in 0..100, (3) every hunk's filePath is
  relative (no leading "/"), and (4) patchSummary is non-empty. On any failure returns
  Guardrail.reject(<structured-error>).

- TestGateRunner.java — pure deterministic logic (no LLM). Inputs: PatchResult and
  PreparedSnapshot. Outputs: GateReport. Applies hunks to the snapshot's text files
  using the line-range metadata in each HunkChange, then runs the bundled in-process
  test stubs (one per seeded task type). Scoring: every stub asserts a specific output
  string is present in the patched file; all stubs PASS → GateVerdict.PASS.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9219 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  PatchEngineerAgent.definition() binds the configured provider.

- src/main/resources/sample-events/seed-tasks.jsonl with 3 seeded tasks:
  a Python AttributeError in a data-pipeline library (12-file snapshot, 3 test files),
  a Go nil-pointer dereference in an HTTP handler (8-file snapshot, 2 test files),
  a TypeScript type-narrowing error in a form validation utility (6-file snapshot, 2
  test files). Each snapshot is a small in-memory representation (file path → content).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (C1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = recommend-only (the agent's patch is advisory, a human
  merges it), oversight.human_in_loop = true, failure.failure_modes including
  "incorrect-patch", "test-evasion", "secret-leakage-via-llm", "out-of-scope-edit";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/patch-engineer.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: SWE-Bench Agent", prerequisites,
  generate-the-system, what-you-get, customize-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of task cards; right = selected-task detail with bug description,
  snapshot file tree, diff viewer, test case table, and gate verdict chip).
  Browser title exactly: <title>Akka Sample: SWE-Bench Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface. Each branch
  reads src/main/resources/mock-responses/patch-issue.json, picks one entry
  pseudo-randomly per call (seedFor(taskId)), and deserialises into PatchResult.
- Per-task mock-response shapes for THIS blueprint:
    patch-issue.json — 6 PatchResult entries covering all three language types.
      Each entry has a valid unifiedDiff block, 1–3 HunkChange records, a
      confidenceScore between 55 and 95, and a one-sentence patchSummary.
      Plus 2 deliberately MALFORMED entries (one with confidenceScore = 150; one with
      a hunk filePath beginning with "/") — PatchGuardrail blocks both, exercising the
      retry path. The mock selects a malformed entry on the FIRST iteration of every
      3rd task (modulo seed) so J2 is reproducible.
    patch-issue.json also includes 1 FAILING-GATE entry that passes structural
      validation but whose diff produces test failures when TestGateRunner applies it.
- MockModelProvider.seedFor(taskId) makes per-task selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PatchEngineerAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. BenchmarkTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (patchStep 120s,
  awaitSnapshotStep 15s, testGateStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on BenchmarkTask is Optional<T>.
- Lesson 7: BenchmarkTasks.java with PATCH_ISSUE is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9219 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides and
  themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (PatchEngineerAgent). The
  TestGateRunner is rule-based and does NOT make an LLM call.
- The snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated patchStep uses TaskDef.attachment(...).
- PatchGuardrail is wired via the agent's guardrail-configuration mechanism.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
