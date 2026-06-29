# SPEC — code-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CodeAssistant.
**One-line pitch:** A developer submits a coding task description and a repository snapshot; one AI agent reads the files (passed as task attachments, never as inline prompt text), may invoke sandboxed analysis tools, and returns a structured `EditPlan` — a list of file changes with before/after diffs, a recommended test command, and a confidence rating.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `CodeAssistantAgent` (AutonomousAgent) carries the entire proposal; the surrounding components prepare its input, enforce tool-call safety, and gate the output through CI. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every tool call the agent makes before execution. Shell commands and file-mutation calls are validated against an allowlist; disallowed calls are rejected with a structured error so the agent can retry without the blocked operation.
- A **CI gate** runs immediately after `EditProposed` lands on the entity. A `CIGate` Consumer subscribes to the event, runs the project's test suite against the proposed file changes, and emits either `GatePassed` or `GateFailed`. The developer only sees a "ready" plan when the gate passes.

The blueprint shows that the single-agent pattern does not mean "unguarded" — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **repository snapshot** from a dropdown (three seeded examples — a Java REST service, a Python data pipeline, a TypeScript CLI tool) or pastes a custom snapshot as a list of file paths and contents.
2. The user types a **task description** (e.g., "Add input validation to the `/submit` endpoint and cover it with tests").
3. The user enters a **task id** label (free text) and their **author** identifier, then clicks **Submit task**.
4. The UI POSTs to `/api/edits` and receives an `editId`.
5. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `ANALYZING` — the agent has received the repository files as attachments.
6. Within ~10–30 s the agent call completes. The card transitions to `EDIT_PROPOSED`. The proposed edit plan appears: a confidence badge (HIGH / MEDIUM / LOW), a brief rationale, a per-file change table (file path, change type, diff), and the recommended test command.
7. Within ~5 s the CI gate finishes. The card transitions to `GATE_PASSED` or `GATE_FAILED`. If passed, a green badge and the test output summary appear. If failed, the test failure output is shown inline so the developer can decide whether to request a revised plan.
8. The user can submit another task; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EditEndpoint` | `HttpEndpoint` | `/api/edits/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `EditEntity`, `EditView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `EditEntity` | `EventSourcedEntity` | Per-task lifecycle: submitted → analyzing → proposed → gate-passed/failed. Source of truth. | `EditEndpoint`, `CIGate`, `EditWorkflow` | `EditView` |
| `CIGate` | `Consumer` | Subscribes to `EditProposed` events; runs test suite against proposed changes; calls `EditEntity.recordGateResult`. | `EditEntity` events | `EditEntity` |
| `EditWorkflow` | `Workflow` | One workflow per editId. Steps: `analyzeStep` → `awaitGateStep`. | started by `EditEndpoint` after submit | `CodeAssistantAgent`, `EditEntity` |
| `CodeAssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the task description as the task instruction and repository files as task attachments; returns `EditPlan`. May call sandboxed analysis tools. | invoked by `EditWorkflow` | returns edit plan |
| `ToolCallGuardrail` | supporting class | Intercepts every tool call before execution; validates against allowlist; rejects disallowed calls. | wired on `CodeAssistantAgent` | agent loop |
| `EditView` | `View` | Read model: one row per edit task for the UI. | `EditEntity` events | `EditEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record FileSnapshot(
    String filePath,
    String content,
    String language
) {}

record RepoSnapshot(
    String snapshotId,
    String projectName,
    List<FileSnapshot> files
) {}

record TaskRequest(
    String editId,
    String taskDescription,
    RepoSnapshot repoSnapshot,
    String author,
    Instant submittedAt
) {}

record FileChange(
    String filePath,
    ChangeType changeType,
    String beforeContent,
    String afterContent,
    String diffSummary
) {}
enum ChangeType { ADD, MODIFY, DELETE }

record EditPlan(
    Confidence confidence,
    String rationale,
    List<FileChange> changes,
    String testCommand,
    Instant proposedAt
) {}
enum Confidence { HIGH, MEDIUM, LOW }

record GateResult(
    GateStatus status,
    String testOutput,
    int testsPassed,
    int testsFailed,
    Instant evaluatedAt
) {}
enum GateStatus { PASSED, FAILED }

record Edit(
    String editId,
    Optional<TaskRequest> request,
    Optional<EditPlan> plan,
    Optional<GateResult> gateResult,
    EditStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EditStatus {
    SUBMITTED, ANALYZING, EDIT_PROPOSED, GATE_PASSED, GATE_FAILED, FAILED
}
```

Events on `EditEntity`: `TaskSubmitted`, `AnalysisStarted`, `EditProposed`, `GatePassed`, `GateFailed`, `EditFailed`.

Every nullable lifecycle field on the `Edit` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/edits` — body `{ taskDescription, repoSnapshot: RepoSnapshot, author }` → `{ editId }`.
- `GET /api/edits` — list all edits, newest-first.
- `GET /api/edits/{id}` — one edit.
- `GET /api/edits/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: CodeAssistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tasks (status pill + confidence badge + age) and a right pane with the selected task's detail — task description, proposed file changes (before/after diff), recommended test command, and CI gate result.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call the `CodeAssistantAgent` attempts. Asserts the tool name is in the allowed set (`read_file`, `list_directory`, `search_code`, `run_tests`). Shell execution commands (`exec`, `run_shell`, `bash`, and any tool whose name contains `shell` or `exec`) are blocked. File mutation tools (`write_file`, `delete_file`, `move_file`) are blocked — the agent proposes changes in the `EditPlan` structure; it does not write files directly. On rejection, returns a structured `tool-blocked` error to the agent loop so the task retries without the disallowed call. This bounds the agent's ability to perform side effects during analysis.
- **C1 — CI gate** (`ci-gate`, `test-gate`): runs inside `CIGate` Consumer, triggered by `EditProposed`. Applies the proposed file changes to an in-memory copy of the repository snapshot, then executes the `testCommand` from the `EditPlan` inside a sandboxed JVM subprocess. Records pass/fail and full test output. Emits `GatePassed` or `GateFailed` on the entity. The developer never receives a "ready" edit plan that fails the project's own tests.

## 9. Agent prompts

- `CodeAssistantAgent` → `prompts/code-assistant.md`. The single decision-making LLM. System prompt instructs it to read the attached repository files, understand the task description, produce a minimal set of file changes that satisfy the task, and return a single `EditPlan`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Java REST service seed with a task asking for input validation; within 30 s the edit plan appears with at least one `MODIFY` change and a passing CI gate badge.
2. **J2** — Agent attempts a `run_shell` call during analysis (mock LLM path) — the `before-tool-call` guardrail blocks it; the next iteration proceeds without the blocked call and produces a valid plan.
3. **J3** — A proposed edit breaks the seeded test suite → CI gate fails → the card shows `GATE_FAILED` with test output; the developer is not handed a broken plan.
4. **J4** — Repository files submitted in the task body never appear as inline instruction text; only the task attachments carry the file contents (verifiable from the agent task log).

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named code-assistant demonstrating the single-agent × dev-code cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-code-assistant. Java package io.akka.samples.codeassistant. Akka 3.6.0.
HTTP port 9765.

Components to wire (exactly):

- 1 AutonomousAgent CodeAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/code-assistant.md>) and
  .capability(TaskAcceptance.of(PROPOSE_EDITS).maxIterationsPerTask(4)). The task receives
  the coding task description as its instruction text and each repository file as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is
  the canonical call, one attachment per FileSnapshot). Output: EditPlan{confidence:
  Confidence (HIGH/MEDIUM/LOW), rationale: String, changes: List<FileChange>, testCommand:
  String, proposedAt: Instant}. The agent is configured with a before-tool-call guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries the tool call within its 4-iteration budget.

- 1 Workflow EditWorkflow per editId with two steps:
  * analyzeStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    CodeAssistantAgent.class, "assistant-" + editId).runSingleTask(
      TaskDef.instructions(request.taskDescription())
        .attachment(<one attachment per FileSnapshot: filePath as name, content as bytes>)
    ) — returns a taskId, then forTask(taskId).result(PROPOSE_EDITS) to fetch the plan.
    On success calls EditEntity.recordPlan(plan). WorkflowSettings.stepTimeout 90s with
    defaultStepRecovery maxRetries(2).failoverTo(EditWorkflow::error).
  * awaitGateStep — polls EditEntity.getEdit every 2s; on edit.gateResult().isPresent()
    advances to completion. WorkflowSettings.stepTimeout 30s (CI gate is in-process).
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity EditEntity (one per editId). State Edit{editId: String,
  request: Optional<TaskRequest>, plan: Optional<EditPlan>, gateResult: Optional<GateResult>,
  status: EditStatus, createdAt: Instant, finishedAt: Optional<Instant>}. EditStatus enum:
  SUBMITTED, ANALYZING, EDIT_PROPOSED, GATE_PASSED, GATE_FAILED, FAILED. Events:
  TaskSubmitted{request}, AnalysisStarted{}, EditProposed{plan}, GatePassed{gateResult},
  GateFailed{gateResult}, EditFailed{reason}. Commands: submit, markAnalyzing, recordPlan,
  recordGateResult, fail, getEdit. emptyState() returns Edit.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer CIGate subscribed to EditEntity events; on EditProposed runs an in-memory
  CI simulation: applies the proposed FileChange list to a copy of the repository snapshot
  (from request.repoSnapshot()), executes the plan's testCommand in a sandboxed subprocess,
  records stdout/stderr, counts passing/failing tests from the output, builds GateResult,
  then calls EditEntity.recordGateResult(gateResult). GateStatus is PASSED if testsFailed==0,
  FAILED otherwise.

- 1 View EditView with row type EditRow (mirrors Edit). Table updater consumes EditEntity
  events. ONE query getAllEdits: SELECT * AS edits FROM edit_view. No WHERE status filter
  — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * EditEndpoint at /api with POST /edits (body
    {taskDescription, repoSnapshot: {snapshotId, projectName, files: [{filePath, content,
    language}]}, author}; mints editId; calls EditEntity.submit; starts EditWorkflow with
    id = "edit-" + editId; returns {editId}), GET /edits (list from getAllEdits, sorted
    newest-first), GET /edits/{id} (one row), GET /edits/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- EditTasks.java declaring one Task<R> constant: PROPOSE_EDITS = Task.name("Propose edits")
  .description("Read the attached repository files and produce an EditPlan for the coding task")
  .resultConformsTo(EditPlan.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records FileSnapshot, RepoSnapshot, TaskRequest, FileChange, ChangeType, EditPlan,
  Confidence, GateResult, GateStatus, Edit, EditStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Reads the tool name from
  the candidate tool call, checks it against the allowlist {read_file, list_directory,
  search_code, run_tests}, and either passes the call through or returns
  Guardrail.reject(<structured tool-blocked error>) to force the agent loop to retry
  without that call.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9765 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CodeAssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/repo-snapshots.jsonl with 3 seeded repository snapshots:
  a 3-file Java REST service (endpoint + service + test), a 3-file Python data pipeline
  (reader + transformer + test), and a 3-file TypeScript CLI tool (parser + main + test).
  Each snapshot has at least one intentional gap (missing validation, unhandled error, no
  null check) that a coding task can target.

- src/main/resources/sample-events/tasks.jsonl with 3 paired coding tasks: one per seeded
  snapshot, describing the specific gap to fix. Each task is scoped to produce 1–3 file
  changes so the CI simulation stays fast.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, C1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.source-code = true,
  decisions.authority_level = propose-only (the agent's plan is advisory; a developer
  applies it), oversight.human_in_loop = true (a developer reads the plan and the gate
  result before merging), failure.failure_modes including "incorrect-change",
  "broken-test-suite", "disallowed-tool-call", "scope-creep", "confidence-miscalibration";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/code-assistant.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: CodeAssistant", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of edit task cards; right = selected-task detail with task description, file
  changes table with diff, test command, and CI gate result).
  Browser title exactly: <title>Akka Sample: CodeAssistant</title>. No subtitle on the
  Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(editId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    propose-edits.json — 6 EditPlan entries covering all three Confidence values.
      Each entry has a rationale paragraph, a changes array with 1–3 FileChange entries
      (each with a non-empty diffSummary and realistic before/after content for the matched
      seeded snapshot), and a testCommand. Plus 2 deliberately MALFORMED entries (one with
      a FileChange whose changeType is outside the enum; one with an empty changes list
      where at least one change is expected) — the guardrail blocks both, exercising the
      retry path. The mock selects a malformed entry on the FIRST iteration of every 3rd
      edit (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(editId) helper makes per-edit selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeAssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion EditTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (analyzeStep
  90s, awaitGateStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on the Edit row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: EditTasks.java with PROPOSE_EDITS = Task.name(...).description(...)
  .resultConformsTo(EditPlan.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9765 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeAssistantAgent).
  The CI gate is rule-based (CIGate Consumer + subprocess) and does NOT make an LLM call
  — keeping the pattern's "one agent" promise honest.
- Repository files are passed as Task ATTACHMENTS, never inlined into the agent's
  instructions. Verify the generated analyzeStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check. Lesson 1's AutonomousAgent contract is the
  authoritative reference for how the hook is registered.
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
