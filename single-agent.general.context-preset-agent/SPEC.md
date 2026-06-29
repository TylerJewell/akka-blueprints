# SPEC — context-preset-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ContextPresetAgent.
**One-line pitch:** A caller supplies an environment and role; one AI agent resolves a matching Request Context Preset, adapts its instructions and model selection accordingly, executes the request using only the tools that preset authorises, and returns a typed `PresetRequestResult`.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `ContextPresetAgent` (AutonomousAgent) carries the entire decision; the surrounding components resolve the preset, store the audit trail, and project the read model. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs inside `ContextPresetAgent` on every tool invocation. It checks the resolved role from the active `PresetDefinition` and rejects any call to `adminActionTool` when the caller's role is not `admin`. The rejection is returned as a structured error; the agent explains to the caller that the action requires admin access and does not retry the blocked call.
- A **CI configuration gate** validates every preset JSON file in `src/main/resources/presets/` against the `PresetDefinition` schema before the artifact is built. A file with a missing `allowedTools` array, an unrecognised `modelId`, or an unrecognised `environment` value fails the build, preventing a misconfigured preset from reaching the agent runtime.

The blueprint shows that a single-agent system can enforce role-based tool boundaries at the framework level rather than relying on the model to self-police.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks an **environment** (`dev` / `staging` / `prod`) and a **role** (`admin` / `guest`) from dropdowns. These are the preset dimensions.
2. The user types a **request** in a freetext field (e.g., "List recent deployments", "Trigger a cache flush", "Show system health").
3. The user clicks **Submit**. The UI POSTs to `/api/preset-requests` and receives a `requestId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s the card transitions to `PRESET_RESOLVED` — the right pane shows the resolved `PresetDefinition` (model id, allowed tools, environment-specific instructions excerpt).
5. Within ~10–30 s the workflow's `executeStep` completes. The card transitions to `EXECUTING` then `COMPLETED`. The result appears: the agent's answer text plus a `toolCallLog` showing which tools were invoked (or a `BLOCKED` entry for any guardrail rejections).
6. If the agent called `adminActionTool` while the role was `guest`, the guardrail blocked it. The card shows a `BLOCKED_TOOL_CALLS` chip listing the blocked calls.
7. The user can submit additional requests. The live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PresetRequestEndpoint` | `HttpEndpoint` | `/api/preset-requests/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PresetRequestEntity`, `PresetRequestView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PresetRequestEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → preset-resolved → executing → completed / failed. Source of truth. | `PresetRequestEndpoint`, `PresetRequestWorkflow` | `PresetRequestView` |
| `PresetRegistry` | `KeyValueEntity` | Stores named `PresetDefinition` objects keyed by `env:role`. Read by the workflow on every request. | seeded at startup | `PresetRequestWorkflow` |
| `PresetRequestWorkflow` | `Workflow` | One workflow per requestId. Steps: `resolvePresetStep` → `executeStep` → `auditStep`. | started by `PresetRequestEndpoint` | `ContextPresetAgent`, `PresetRequestEntity` |
| `ContextPresetAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the resolved preset's instructions as the system prompt addition, executes the caller's request using the preset's allowed tools, and returns `PresetRequestResult`. | invoked by `PresetRequestWorkflow` | returns result |
| `PresetRequestView` | `View` | Read model: one row per request for the UI. | `PresetRequestEntity` events | `PresetRequestEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PresetDefinition(
    String presetId,           // "<env>:<role>"
    String environment,        // "dev" | "staging" | "prod"
    String role,               // "admin" | "guest"
    String modelId,            // e.g. "claude-sonnet-4-6"
    List<String> allowedTools, // tool names this preset may invoke
    String instructionAddendum // additional system-prompt text for this preset
) {}

record PresetRequest(
    String requestId,
    String environment,
    String role,
    String requestText,
    String submittedBy,
    Instant submittedAt
) {}

record ToolCallLogEntry(
    String toolName,
    ToolCallStatus status,   // INVOKED | BLOCKED
    String inputSummary,
    String outputSummary,
    Instant calledAt
) {}
enum ToolCallStatus { INVOKED, BLOCKED }

record PresetRequestResult(
    String answerText,
    List<ToolCallLogEntry> toolCallLog,
    Instant completedAt
) {}

record AuditEntry(
    String requestId,
    String presetId,
    int toolCallCount,
    int blockedToolCallCount,
    Instant auditedAt
) {}

record PresetRequestState(
    String requestId,
    Optional<PresetRequest> request,
    Optional<PresetDefinition> resolvedPreset,
    Optional<PresetRequestResult> result,
    Optional<AuditEntry> audit,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    SUBMITTED, PRESET_RESOLVED, EXECUTING, COMPLETED, FAILED
}
```

Events on `PresetRequestEntity`: `RequestSubmitted`, `PresetResolved`, `ExecutionStarted`, `RequestCompleted`, `RequestAudited`, `RequestFailed`.

Every nullable lifecycle field on `PresetRequestState` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/preset-requests` — body `{ environment, role, requestText, submittedBy }` → `{ requestId }`.
- `GET /api/preset-requests` — list all requests, newest-first.
- `GET /api/preset-requests/{id}` — one request.
- `GET /api/preset-requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ContextPresetAgent</title>`.

The App UI tab is a two-column layout: a left rail with the submission form and the live list of submitted requests (status pill + preset badge + age) and a right pane with the selected request's detail — resolved preset definition, agent answer text, tool call log with INVOKED / BLOCKED chips per entry, and audit summary.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call attempted by `ContextPresetAgent`. Reads the resolved `PresetDefinition.allowedTools` list from the active task context. If the attempted tool name is not in `allowedTools`, returns `Guardrail.reject(structured-error)` with a clear reason. The agent receives the rejection, explains to the caller that the action is not permitted for their role, and does not retry the blocked call. The rejection is recorded in `ToolCallLogEntry.status = BLOCKED` and lands in the entity's `RequestCompleted` event payload.
- **C1 — CI configuration gate**: a lightweight validator script at `tools/validate-presets.sh` reads every `src/main/resources/presets/*.json` file, validates each against the `PresetDefinition` schema (required fields: `presetId`, `environment`, `role`, `modelId`, `allowedTools`, `instructionAddendum`; `environment` in `{dev,staging,prod}`; `role` in `{admin,guest}`; `modelId` in the approved model list), and exits non-zero on any violation. The Maven build wires this as a `validate` lifecycle phase via the `exec-maven-plugin` — the artifact cannot be built from a tree containing invalid preset files.

## 9. Agent prompts

- `ContextPresetAgent` → `prompts/context-preset-agent.md`. The single decision-making LLM. System prompt instructs it to treat the resolved preset's `instructionAddendum` as authoritative context for this session, attempt tool calls only for the task at hand, and return a `PresetRequestResult` with a clear answer and a full tool call log.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — `prod` + `admin` caller requests a cache flush; `adminActionTool` executes; `toolCallLog` shows `INVOKED`.
2. **J2** — `prod` + `guest` caller requests the same action; the `before-tool-call` guardrail blocks `adminActionTool`; the card shows a `BLOCKED_TOOL_CALLS` chip; the tool call log shows `BLOCKED`; `adminActionTool` never fires.
3. **J3** — A preset JSON file with `"environment": "production"` (not in the allowed set) fails the CI configuration gate; the Maven build exits non-zero before any artifact is produced.
4. **J4** — A `dev` + `admin` caller gets `diagnosticTool` in their resolved preset's `allowedTools`; a `prod` + `admin` caller's resolved preset does not include `diagnosticTool`; the prod caller's tool call log shows `BLOCKED` if `diagnosticTool` is attempted.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named context-preset-agent demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-context-preset-agent. Java package
io.akka.samples.requestcontextpresetsagent. Akka 3.6.0. HTTP port 9475.

Components to wire (exactly):

- 1 AutonomousAgent ContextPresetAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/context-preset-agent.md>) and
  .capability(TaskAcceptance.of(EXECUTE_PRESET_REQUEST).maxIterationsPerTask(3)). The task
  receives the resolved preset's instructionAddendum and the caller's requestText in the
  task instructions text. The agent has access to three simulated tools: readStateTool
  (available to all presets), adminActionTool (admin role only), and diagnosticTool
  (dev/staging only). The agent is configured with a before-tool-call guardrail (see G1 in
  eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection for adminActionTool the agent does NOT retry — it explains the restriction and
  returns a completed result. Output: PresetRequestResult{answerText: String, toolCallLog:
  List<ToolCallLogEntry>, completedAt: Instant}.

- 1 Workflow PresetRequestWorkflow per requestId with three steps:
  * resolvePresetStep — calls PresetRegistry.getPreset(env + ":" + role); on
    result.isPresent() emits PresetResolved and advances to executeStep.
    WorkflowSettings.stepTimeout 5s.
  * executeStep — emits ExecutionStarted, then calls componentClient.forAutonomousAgent(
    ContextPresetAgent.class, "agent-" + requestId).runSingleTask(
      TaskDef.instructions(buildInstructions(resolved, request))
    ) — returns a taskId, then forTask(taskId).result(EXECUTE_PRESET_REQUEST) to fetch
    the result. On success calls PresetRequestEntity.complete(result). On failure calls
    PresetRequestEntity.fail(reason). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(PresetRequestWorkflow::error).
  * auditStep — computes AuditEntry from the completed result (counts INVOKED vs BLOCKED
    entries in toolCallLog); calls PresetRequestEntity.recordAudit(auditEntry).
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PresetRequestEntity (one per requestId). State PresetRequestState
  {requestId: String, request: Optional<PresetRequest>, resolvedPreset:
  Optional<PresetDefinition>, result: Optional<PresetRequestResult>, audit:
  Optional<AuditEntry>, status: RequestStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RequestStatus enum: SUBMITTED, PRESET_RESOLVED,
  EXECUTING, COMPLETED, FAILED. Events: RequestSubmitted{request}, PresetResolved{preset},
  ExecutionStarted{}, RequestCompleted{result}, RequestAudited{audit}, RequestFailed{reason}.
  Commands: submit, resolvePreset, markExecuting, complete, recordAudit, fail, getRequest.
  emptyState() returns PresetRequestState.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 KeyValueEntity PresetRegistry keyed by "<env>:<role>". State PresetDefinition. Commands:
  upsertPreset, getPreset, deletePreset. Seeded at startup via Bootstrap.java which calls
  upsertPreset for each entry in src/main/resources/presets/. Six seed presets: dev:admin,
  dev:guest, staging:admin, staging:guest, prod:admin, prod:guest. Each carries a
  distinct modelId, allowedTools list, and instructionAddendum.

- 1 View PresetRequestView with row type PresetRequestRow (mirrors PresetRequestState
  minus raw tool output verbatim — the view holds summaries). Table updater consumes
  PresetRequestEntity events. ONE query getAllRequests: SELECT * AS requests FROM
  preset_request_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PresetRequestEndpoint at /api with POST /preset-requests (body {environment, role,
    requestText, submittedBy}; mints requestId; calls PresetRequestEntity.submit; starts
    PresetRequestWorkflow; returns {requestId}), GET /preset-requests (list from
    getAllRequests, sorted newest-first), GET /preset-requests/{id} (one row), GET
    /preset-requests/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- PresetRequestTasks.java declaring one Task<R> constant: EXECUTE_PRESET_REQUEST =
  Task.name("Execute preset request").description("Resolve the caller preset, execute the
  request using allowed tools, and return a PresetRequestResult").resultConformsTo(
  PresetRequestResult.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records PresetDefinition, PresetRequest, ToolCallLogEntry, ToolCallStatus,
  PresetRequestResult, AuditEntry, PresetRequestState, RequestStatus.

- ToolGatingGuardrail.java implementing the before-tool-call hook. Reads the resolved
  PresetDefinition from the task context, checks whether the attempted tool name is present
  in allowedTools, and either passes the call through or returns Guardrail.reject(
  structured-error naming the tool and the active role) to block it. The rejection is NOT
  a retry signal — the agent receives it as a permission-denied explanation.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9475 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ContextPresetAgent.definition()
  overrides modelId per-task from the resolved PresetDefinition.modelId when the preset
  specifies a configured provider model.

- src/main/resources/presets/ with 6 preset JSON files (dev-admin.json, dev-guest.json,
  staging-admin.json, staging-guest.json, prod-admin.json, prod-guest.json). Each carries
  presetId, environment, role, modelId, allowedTools, instructionAddendum.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, C1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for the general domain (sector: general,
  decisions.authority_level: execute-with-constraints, capabilities.predictive-policing:
  false, oversight.human_in_loop: false); deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/context-preset-agent.md loaded as the agent system prompt.

- README.md at the project root (matches root blueprint README).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of request cards; right = selected-request detail with resolved
  preset, agent answer, tool call log with INVOKED/BLOCKED chips, audit summary).
  Browser title exactly: <title>Akka Sample: ContextPresetAgent</title>. No subtitle on
  the Overview tab.

- tools/validate-presets.sh — the CI configuration gate script. Reads every
  src/main/resources/presets/*.json file, validates presence of all required fields,
  validates environment in {dev,staging,prod}, role in {admin,guest}, modelId in the
  approved model list {claude-sonnet-4-6, gpt-4o, gemini-2.5-flash, mock}, and exits
  non-zero with a descriptive message on any violation. Wired as Maven validate phase
  via exec-maven-plugin in pom.xml so the build fails before compile if any preset is
  malformed.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    execute-preset-request.json — 8 PresetRequestResult entries. Each entry has an
      answerText paragraph and a toolCallLog array. Entries cover:
        - prod:admin: readStateTool INVOKED + adminActionTool INVOKED (2 entries)
        - prod:guest: readStateTool INVOKED + adminActionTool BLOCKED (2 entries)
        - dev:admin: readStateTool INVOKED + diagnosticTool INVOKED + adminActionTool
          INVOKED (2 entries)
        - dev:guest: readStateTool INVOKED only (1 entry)
        - prod:admin with a BLOCKED entry for a non-existent tool (1 entry, exercises
          the guardrail path)
- A MockModelProvider.seedFor(requestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ContextPresetAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PresetRequestTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (resolvePresetStep
  5s, executeStep 60s, auditStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the PresetRequestState row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: PresetRequestTasks.java with EXECUTE_PRESET_REQUEST = Task.name(...).description(...)
  .resultConformsTo(PresetRequestResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9475 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ContextPresetAgent).
  The audit step in the workflow is deterministic arithmetic over the tool call log —
  no LLM call — keeping the pattern's "one agent" promise honest.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external post-hoc check. The rejection reaches the agent loop before the tool
  executes. Lesson 1's AutonomousAgent contract is the authoritative reference for how the
  hook is registered.
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
