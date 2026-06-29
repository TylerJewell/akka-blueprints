# SPEC — multi-provider-router

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MultiProviderRouter.
**One-line pitch:** A user submits a prompt; one AI agent dispatches it through a load-balancing router that shuffles across OpenAI `gpt-4o-mini` and Anthropic Claude, records the call's latency and provider identity, and returns the response alongside a running quality dashboard.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `RouterAgent` (AutonomousAgent) carries every dispatch; the surrounding components only route its input, log its output, and evaluate aggregate trends. One governance mechanism is wired around the agent:

- A **periodic performance evaluator** runs on a time-window basis (every N completed calls) over the accumulated `CallEntity` records. It computes per-provider latency percentiles, response-quality scores, and error rates, then writes an `EvalReport` that the UI surfaces on the Eval Matrix tab. The evaluator is deterministic and rule-based — not an LLM call — so the single-agent invariant is preserved.

The blueprint shows that a load-balancing agent can be governed by continuous comparative monitoring without adding a second decision-making model to the system.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a prompt into the **Prompt** textarea (or picks one of three seeded examples — a factual question, a summarization task, a code-generation request).
2. (Optional) The user pins the **Provider** to one of `auto` (shuffle), `openai`, or `anthropic`, or leaves it on `auto`.
3. The user clicks **Send**. The UI POSTs to `/api/calls` and receives a `callId`.
4. The card appears in the live list in `PENDING` state. Within ~1–30 s (depending on provider latency), the card transitions to `COMPLETED` and shows: the selected provider, the response text, and the round-trip latency in ms.
5. If the call fails (timeout, provider error), the card transitions to `FAILED` and shows an error reason.
6. Periodically (every 5 completed calls), the `PerformanceMonitor` Consumer triggers an `EvalReport`. The Eval Matrix tab updates with per-provider aggregates.
7. The user can submit more prompts; the live list keeps history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RouterEndpoint` | `HttpEndpoint` | `/api/calls/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `CallEntity`, `CallView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CallEntity` | `EventSourcedEntity` | Per-call lifecycle: pending → dispatched → completed / failed. Source of truth. | `RouterEndpoint`, `RoutingWorkflow` | `CallView` |
| `RoutingWorkflow` | `Workflow` | One workflow per call. Steps: `dispatchStep` → `recordStep`. | started by `RouterEndpoint` after minting callId | `RouterAgent`, `CallEntity` |
| `RouterAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the prompt as the task instruction; the LiteLLMRouterModel selects the provider. Returns `CallResult`. | invoked by `RoutingWorkflow` | returns result |
| `PerformanceMonitor` | `Consumer` | Subscribes to `CallCompleted` events; when the call count crosses a window threshold, runs `EvaluationAggregator` and writes an `EvalReport` to `MonitorView`. | `CallEntity` events | `MonitorView` |
| `CallView` | `View` | Read model: one row per call for the UI. | `CallEntity` events | `RouterEndpoint` |
| `MonitorView` | `View` | Read model: one row per eval window per provider. | `PerformanceMonitor` writes | `RouterEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum Provider { OPENAI, ANTHROPIC, MOCK }
enum ProviderHint { AUTO, OPENAI, ANTHROPIC }
enum CallStatus { PENDING, DISPATCHED, COMPLETED, FAILED }

record CallRequest(
    String callId,
    String promptText,
    ProviderHint hint,
    String submittedBy,
    Instant submittedAt
) {}

record CallResult(
    Provider selectedProvider,
    String responseText,
    long latencyMs,
    Instant completedAt
) {}

record CallRecord(
    String callId,
    Optional<CallRequest> request,
    Optional<CallResult> result,
    CallStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    Optional<String> errorReason
) {}

record ProviderStats(
    Provider provider,
    int totalCalls,
    int failedCalls,
    long p50LatencyMs,
    long p95LatencyMs,
    double avgQualityScore   // 1.0–5.0, rule-based
) {}

record EvalReport(
    String windowId,
    int windowSize,
    List<ProviderStats> stats,
    Instant generatedAt
) {}

record MonitorRow(
    String windowId,
    int windowSize,
    List<ProviderStats> stats,
    Instant generatedAt
) {}
```

Events on `CallEntity`: `CallSubmitted`, `CallDispatched`, `CallCompleted`, `CallFailed`.

Every nullable lifecycle field on `CallRecord` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/calls` — body `{ promptText, hint: "AUTO"|"OPENAI"|"ANTHROPIC", submittedBy }` → `{ callId }`.
- `GET /api/calls` — list all call records, newest-first.
- `GET /api/calls/{id}` — one call record.
- `GET /api/calls/sse` — Server-Sent Events; one event per state transition.
- `GET /api/monitor` — list all eval reports, newest-first.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MultiProviderRouter</title>`.

The App UI tab is a two-column layout: a left rail with the submission panel and the live call list (provider badge + status pill + latency chip + age) and a right pane with the selected call's detail — prompt text, provider selected, response text, latency, and (when available) the eval-window quality score for that provider.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — periodic performance evaluator** (`eval-periodic`, `performance-monitor`): runs inside `PerformanceMonitor` Consumer every time 5 `CallCompleted` events have accumulated. The evaluator (`EvaluationAggregator`) computes per-provider latency percentiles (P50, P95), error rate, and a rule-based quality score (1–5) derived from response length relative to prompt length, absence of truncation markers, and non-empty response body. Writes an `EvalReport`. Non-blocking: individual calls are not gated on the evaluation; the report is advisory. The monitor view surfaces the latest report per provider on the Eval Matrix tab.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. The single dispatch LLM. System prompt instructs it to answer the prompt faithfully, identify its own provider identity in a structured header, and return a `CallResult`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a factual prompt with hint `AUTO`; within 30 s the call completes, the provider badge appears, and latency is visible.
2. **J2** — User submits 6 prompts; the live list shows calls distributed across both providers; neither provider handles all 6.
3. **J3** — After 5 completed calls the Eval Matrix tab shows a new eval report with per-provider stats; P50/P95 latency values are non-zero.
4. **J4** — User pins hint to `OPENAI` and submits 2 calls; both land on `OPENAI`; the router respects the hint.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-provider-router demonstrating the single-agent × general cell.
Runs out of the box (no external services in mock mode). Maven group io.akka.samples.
Maven artifact single-agent-general-multi-provider-router. Java package
io.akka.samples.multillmloadbalancedagent. Akka 3.6.0. HTTP port 9671.

Components to wire (exactly):

- 1 AutonomousAgent RouterAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/router-agent.md>) and
  .capability(TaskAcceptance.of(ROUTE_PROMPT).maxIterationsPerTask(2)). The task receives
  the prompt text as its instruction text and the caller-supplied ProviderHint via a task
  metadata field. The LiteLLMRouterModel is bound in application.conf as the agent's
  model-provider; the simple-shuffle strategy distributes calls across the two configured
  backends. Output: CallResult{selectedProvider: Provider, responseText: String,
  latencyMs: long, completedAt: Instant}. The companion RouterTasks.java MUST exist
  (Lesson 7).

- 1 Workflow RoutingWorkflow per callId with two steps:
  * dispatchStep — emits CallDispatched, then calls componentClient.forAutonomousAgent(
    RouterAgent.class, "router-" + callId).runSingleTask(
      TaskDef.instructions(callRequest.promptText())
    ) — returns a taskId, then forTask(taskId).result(ROUTE_PROMPT) to fetch CallResult.
    On success calls CallEntity.complete(result). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(1).failoverTo(RoutingWorkflow::error).
  * recordStep — no-op step that verifies CallEntity reached COMPLETED; transitions to
    done. WorkflowSettings.stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity CallEntity (one per callId). State CallRecord{callId: String,
  request: Optional<CallRequest>, result: Optional<CallResult>, status: CallStatus,
  createdAt: Instant, finishedAt: Optional<Instant>, errorReason: Optional<String>}.
  CallStatus enum: PENDING, DISPATCHED, COMPLETED, FAILED. Events: CallSubmitted{request},
  CallDispatched{}, CallCompleted{result}, CallFailed{reason}. Commands: submit, dispatch,
  complete, fail, getCall. emptyState() returns CallRecord.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer PerformanceMonitor subscribed to CallEntity events; on CallCompleted
  increments an in-memory window counter (or reads the count from MonitorView). When the
  window counter reaches 5 (configurable), runs EvaluationAggregator over the last N
  CallRecord rows fetched from CallView, computes ProviderStats per Provider (P50 latency,
  P95 latency, error rate, quality score), builds EvalReport, and writes a MonitorRow to
  MonitorView via a direct upsert. Window id = "window-" + (totalCompletedCalls / 5).

- 2 Views:
  * CallView with row type CallRow (mirrors CallRecord). Table updater consumes CallEntity
    events. ONE query getAllCalls: SELECT * AS calls FROM call_view. No WHERE status
    filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.
  * MonitorView with row type MonitorRow. Table updater accepts upsert calls from
    PerformanceMonitor. ONE query getAllReports: SELECT * AS reports FROM monitor_view.

- 2 HttpEndpoints:
  * RouterEndpoint at /api with POST /calls (body {promptText, hint: "AUTO"|"OPENAI"|
    "ANTHROPIC", submittedBy}; mints callId; calls CallEntity.submit; starts
    RoutingWorkflow; returns {callId}), GET /calls (list from getAllCalls, newest-first),
    GET /calls/{id} (one row), GET /calls/sse (Server-Sent Events forwarded from CallView
    stream-updates), GET /monitor (list from getAllReports, newest-first), and three
    /api/metadata/* endpoints serving YAML/MD from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- RouterTasks.java declaring one Task<R> constant: ROUTE_PROMPT = Task.name("Route
  prompt").description("Dispatch the prompt to the configured provider and return a
  CallResult").resultConformsTo(CallResult.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records CallRequest, CallResult, CallRecord, ProviderStats, EvalReport, MonitorRow,
  and enums Provider, ProviderHint, CallStatus.

- EvaluationAggregator.java — pure deterministic logic (no LLM). Inputs: List<CallRow> for
  a given window, grouped by Provider. Outputs: List<ProviderStats>. Latency percentiles
  computed over the window's latencyMs values; quality score: 5 if response is non-empty
  and >50 chars with no truncation markers, 3 if short but non-empty, 1 if empty. Scoring
  rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9671, the
  LiteLLMRouterModel configuration block pointing to the two backends (anthropic
  claude-sonnet-4-6 and openai gpt-4o-mini) reading ${?ANTHROPIC_API_KEY} and
  ${?OPENAI_API_KEY}, and a mock fallback. The RouterAgent.definition() binds the router
  via the configured model-provider pattern from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded prompts: a factual
  question (What is the capital of Portugal?), a summarization task (5-sentence summary of
  a short paragraph), and a code-generation request (write a Java method that reverses a
  string).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the metadata endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching Section 8.
  Matching simplified_view list. No regulation_anchors — general domain.

- risk-survey.yaml at the project root with decisions.authority_level = informational
  (the agent returns responses, not binding decisions), oversight.human_in_loop = false
  (fully automated dispatch), failure.failure_modes including "provider-unavailable",
  "latency-spike", "response-truncation", "provider-quality-divergence";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/router-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Multi-LLM Load-Balanced Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission panel + live call cards; right = selected-call detail with prompt, provider
  badge, response text, latency chip, and eval-window quality score chip for that provider).
  Browser title exactly: <title>Akka Sample: MultiProviderRouter</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY and OPENAI_API_KEY. If both are set, default to LiteLLMRouterModel
  with both backends and proceed silently. If only one is set, configure that provider
  as primary and warn that load balancing requires both keys.
- If neither is set, ask the user how to source the keys, offering five options:
    (a) Mock LLM — no real keys; generate MockModelProvider returning random-but-shape-
        correct CallResult outputs. Sets model-provider = mock. The mock alternates between
        Provider.OPENAI and Provider.ANTHROPIC deterministically by callId hash.
    (b) Name existing env vars — record the env-var NAMES in application.conf; /akka:build
        forwards values from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — values live only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  references if they do not resolve at runtime. The message must not echo any captured
  key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  dispatch on the ROUTE_PROMPT task id. The mock reads
  src/main/resources/mock-responses/route-prompt.json, picks one entry using
  MockModelProvider.seedFor(callId) (deterministic, reproducible), and deserialises into
  CallResult. The mock alternates Provider.OPENAI and Provider.ANTHROPIC on even/odd
  seeds so distribution tests (J2) are reproducible.
- route-prompt.json includes 10 CallResult entries: 5 with Provider.OPENAI and 5 with
  Provider.ANTHROPIC. Latency values vary (200–1800 ms). Response texts cover the three
  seed prompt types. 2 entries have short responseText (<20 chars) to exercise the
  quality scorer's low-score path. All entries have completedAt = "2026-06-28T00:00:00Z"
  (overridden at runtime with the actual Instant).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. RouterAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RouterTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (dispatchStep
  60s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on CallRecord is Optional<T>. The view table
  updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: RouterTasks.java with ROUTE_PROMPT = Task.name(...).description(...)
  .resultConformsTo(CallResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o-mini. No
  deprecated model names.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9671 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (RouterAgent). The
  periodic evaluator is rule-based (EvaluationAggregator.java) and does NOT make an LLM
  call — keeping the pattern's "one agent" promise honest.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API keys (offer the five valid sourcing options; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
