# SPEC — workflow-observability

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Workflow Observability (Arize/Langfuse).
**One-line pitch:** A continuous worker executes agent workflows, captures full execution traces, and exports them to Arize Phoenix or Langfuse for runtime governance monitoring.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered around a two-agent pipeline (`RouterAgent` + `SummaryAgent`). Specifically:

- A **deployer-runtime-monitoring** mechanism: the `TraceExporter` Consumer intercepts every span event and forwards structured traces to an external observability platform (Arize Phoenix, Langfuse) or the built-in structured log. Deployers fulfil their runtime monitoring obligations without modifying agent code — the trace emission is wired at the infrastructure layer.
- An **eval-periodic** sampler running every 30 minutes scores a sample of completed workload items on output quality and routing accuracy, producing a continuous quality signal that is independent of per-item processing.

The result is a system where every agent execution is traceable end-to-end, and quality trends surface automatically on a regular cadence.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live trace feed: every workload item received, its routing decision, the SummaryAgent output, and accumulated span telemetry.
2. `WorkloadPoller` (TimedAction) ticks every 20 s and inserts new simulated workload items into `WorkloadQueue`. (A `RequestSimulator` style — drips canned items.)
3. For each new item: `TraceCollector` (Consumer) opens a parent span, then `ObservabilityWorkflow` routes it via `RouterAgent` and processes it via `SummaryAgent`.
4. After processing, `TraceExporter` (Consumer) picks up `TracingSpan` events from the entity and pushes them to the configured backend (Arize Phoenix endpoint, Langfuse endpoint, or built-in log).
5. `EvalSampler` (TimedAction) ticks every 30 minutes, picks N completed items without an `evalScore`, calls `EvalJudge`, and writes a score back via an `EvalCompleted` event.
6. The UI surfaces latency histograms, token-count totals, routing accuracy, and per-item eval scores in real time via SSE.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WorkloadPoller` | `TimedAction` | Drips simulated workload items into `WorkloadQueue` every 20 s. | scheduler | `WorkloadQueue` |
| `WorkloadQueue` | `EventSourcedEntity` | Append-only log of `WorkloadItemReceived` events. | `WorkloadPoller`, `ObservabilityEndpoint` | `TraceCollector` |
| `TraceCollector` | `Consumer` | Reads `WorkloadItemReceived` events, opens a root span, and emits `ItemRegistered` on the per-item `WorkloadItemEntity`. | `WorkloadQueue` events | `WorkloadItemEntity` |
| `RouterAgent` | `Agent` (typed, not autonomous) | Classifies the workload item into a routing category (`summarise`, `escalate`, `skip`). | invoked by Workflow | returns `RoutingDecision` |
| `SummaryAgent` | `AutonomousAgent` | Processes the workload item and produces a structured `SummaryResult`. | invoked by Workflow | returns `SummaryResult` |
| `ObservabilityWorkflow` | `Workflow` | Per-item orchestration: route → (if summarise) process → export spans → done. | `TraceCollector` (one workflow per `ItemRegistered`) | `WorkloadItemEntity` |
| `WorkloadItemEntity` | `EventSourcedEntity` | Lifecycle per item: queued → routed → processed → exported → evaluated. Accumulates `TracingSpan` events. | `ObservabilityWorkflow` | `ObservabilityView` |
| `TraceExporter` | `Consumer` | Reads `TracingSpan` events; forwards to Arize Phoenix endpoint, Langfuse endpoint, or built-in structured log based on `TRACE_BACKEND` env var. | `WorkloadItemEntity` events | external observability backend |
| `ObservabilityView` | `View` | Read-model row per item for the UI. | `WorkloadItemEntity` events | `ObservabilityEndpoint` |
| `EvalSampler` | `TimedAction` | Every 30 min, samples completed items without `evalScore`; calls `EvalJudge`; writes `EvalCompleted`. | scheduler | `WorkloadItemEntity` |
| `ObservabilityEndpoint` | `HttpEndpoint` | `/api/traces/*` — list, get, SSE, submit custom item. | — | `ObservabilityView`, `WorkloadItemEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record WorkloadItem(String itemId, String workloadType, String payload, String sourceTag, Instant receivedAt) {}

record TracingSpan(
    String spanId,
    String parentSpanId,
    String operationName,
    Instant startedAt,
    Instant finishedAt,
    long durationMs,
    int promptTokens,
    int completionTokens,
    SpanStatus status,
    String errorMessage
) {}
enum SpanStatus { OK, ERROR, TIMEOUT }

record RoutingDecision(RoutingCategory category, String confidence, String rationale) {}
enum RoutingCategory { SUMMARISE, ESCALATE, SKIP }

record SummaryResult(String summary, String keyFindings, int outputTokens, Instant processedAt) {}

record EvalResult(int score, String rationale, Instant evaluatedAt) {}

record WorkloadItemState(
    String itemId,
    WorkloadItem item,
    Optional<RoutingDecision> routing,
    Optional<SummaryResult> result,
    List<TracingSpan> spans,
    Optional<EvalResult> eval,
    ItemStatus status,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum ItemStatus {
    QUEUED, ROUTED, PROCESSING, EXPORTED, EVALUATED, ESCALATED, SKIPPED, FAILED
}
```

Events on `WorkloadItemEntity`: `ItemRegistered`, `ItemRouted`, `ItemProcessed`, `SpanEmitted`, `TracesExported`, `EvalCompleted`, `ItemEscalated`, `ItemSkipped`, `ItemFailed`.

Events on `WorkloadQueue`: `WorkloadItemReceived` (the original item appended as an audit record).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/traces` — list all items. Optional `?status=…&workloadType=…`.
- `GET /api/traces/{id}` — one item with full span list.
- `POST /api/traces` — submit a custom workload item (body `{ workloadType, payload, sourceTag }`).
- `GET /api/traces/{id}/spans` — span list for one item.
- `GET /api/traces/sse` — Server-Sent Events for every item state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Workflow Observability</title>`.

App UI tab is the most distinctive: it shows the **live trace feed** on the left (items newest-first with status pill and routing chip) and the **selected item detail** on the right (span waterfall, routing rationale, summary output, eval score).

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **M1 — deployer-runtime-monitoring** (`hotl`, applied in `TraceExporter` Consumer): every agent span — routing call, LLM completion, workflow step — is captured as a `TracingSpan` event and forwarded to the configured backend. Deployers satisfy runtime monitoring obligations without code changes to agents.
- **E1 — eval-periodic** (every 30 minutes): scores completed items on output quality (relevance, factual grounding, routing accuracy). Surfaces a rolling quality signal separate from per-item traces.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. Typed classifier. Always returns one of the three `RoutingCategory` values.
- `SummaryAgent` → `prompts/summary-agent.md`. Produces a `SummaryResult` from the workload item payload.
- `EvalJudge` (used by `EvalSampler`) → `prompts/eval-judge.md`. Scores a completed item on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a workload item; it appears in the UI within 20 s; passes route → process → span export; status reaches EXPORTED.
2. **J2** — The trace detail view shows at least two spans: one for `RouterAgent` and one for `SummaryAgent`, with non-zero `durationMs`.
3. **J3** — An item with routing category ESCALATE reaches status ESCALATED; no `SummaryAgent` call is made.
4. **J4** — EvalSampler scores at least one EXPORTED item within 30 minutes; score and rationale appear in the UI.
5. **J5** — A custom item submitted via `POST /api/traces` follows the same pipeline as a polled item.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named workflow-observability demonstrating the continuous-monitor ×
governance-risk cell. Runs out of the box (in-memory workload source; Arize Phoenix and
Langfuse are optional via env vars). Maven group io.akka.samples, artifact
continuous-monitor-governance-risk-workflow-observability. Java package
io.akka.samples.workflowobservabilityarizelangfuse. HTTP port 9281.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RouterAgent — classifier. System prompt loaded from
  prompts/router-agent.md. Input: WorkloadItem{itemId, workloadType, payload, sourceTag,
  receivedAt}. Output: RoutingDecision{category: RoutingCategory (enum
  SUMMARISE/ESCALATE/SKIP), confidence: "high"|"medium"|"low", rationale: String}. Defaults
  to ESCALATE under uncertainty.

- 1 AutonomousAgent SummaryAgent — definition() with capability(TaskAcceptance.of(PROCESS)
  .maxIterationsPerTask(3)). System prompt from prompts/summary-agent.md. Input: WorkloadItem.
  Output: SummaryResult{summary: String, keyFindings: String, outputTokens: int,
  processedAt: Instant}. The agent NEVER calls external APIs; it operates only on the
  provided payload.

- 1 AutonomousAgent EvalJudge — definition() with capability(TaskAcceptance.of(EVALUATE)
  .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md. Input: WorkloadItem +
  SummaryResult. Output: EvalResult{score: Integer 1–5, rationale: String,
  evaluatedAt: Instant}.

- 1 Workflow ObservabilityWorkflow per item with steps:
  routeStep -> conditional branch:
    SUMMARISE -> processStep -> exportStep -> done;
    ESCALATE -> ends with ItemEscalated;
    SKIP -> ends with ItemSkipped.
  routeStep wraps RouterAgent with WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(10)).
  processStep wraps SummaryAgent with stepTimeout 30s. exportStep emits TracesExported and
  completes. On timeout in routeStep, emit ItemFailed.

- 2 EventSourcedEntities:
  * WorkloadQueue — append-only audit log. Command receive(WorkloadItem) emits
    WorkloadItemReceived{item}.
  * WorkloadItemEntity (one per itemId) — full per-item lifecycle. State
    WorkloadItemState{itemId, item: WorkloadItem{itemId, workloadType, payload, sourceTag,
    receivedAt}, Optional<RoutingDecision> routing, Optional<SummaryResult> result,
    List<TracingSpan> spans, Optional<EvalResult> eval, ItemStatus status,
    Instant createdAt, Optional<Instant> completedAt}. ItemStatus enum: QUEUED, ROUTED,
    PROCESSING, EXPORTED, EVALUATED, ESCALATED, SKIPPED, FAILED. Events: ItemRegistered,
    ItemRouted, ItemProcessed, SpanEmitted, TracesExported, EvalCompleted, ItemEscalated,
    ItemSkipped, ItemFailed. Commands: registerItem, recordRouting, recordResult,
    emitSpan, markExported, markEscalated, markSkipped, markFailed, recordEval,
    getItem. emptyState() returns WorkloadItemState.initial() without commandContext().

- 2 Consumers:
  * TraceCollector subscribed to WorkloadQueue events; for each WorkloadItemReceived,
    calls WorkloadItemEntity.registerItem and starts an ObservabilityWorkflow with itemId
    as the workflow id.
  * TraceExporter subscribed to WorkloadItemEntity SpanEmitted events; for each span,
    checks TRACE_BACKEND env var:
      "arize" -> HTTP POST to ARIZE_PHOENIX_URL/api/v1/spans with OTLP-JSON body.
      "langfuse" -> HTTP POST to LANGFUSE_HOST/api/public/ingestion with Langfuse batch body.
      default -> structured log line at INFO level (always active, even when external backend set).
    If the external POST fails, logs the error and continues (non-blocking).

- 1 View ObservabilityView with row type WorkloadItemRow (mirrors WorkloadItemState, includes
  spans list and eval). Table updater consumes WorkloadItemEntity events. ONE query
  getAllItems SELECT * AS items FROM observability_view. No WHERE filter — caller filters
  client-side.

- 2 TimedActions:
  * WorkloadPoller — every 20s, reads next line from
    src/main/resources/sample-events/workload-items.jsonl and calls WorkloadQueue.receive.
  * EvalSampler — every 30 minutes, queries ObservabilityView.getAllItems, picks up to 5
    EXPORTED items without an evalScore (oldest-first), calls EvalJudge per item, then calls
    WorkloadItemEntity.recordEval(score, rationale, evaluatedAt) per item.

- 2 HttpEndpoints:
  * ObservabilityEndpoint at /api with GET /traces (optional ?status, ?workloadType),
    GET /traces/{id}, GET /traces/{id}/spans, POST /traces (body {workloadType, payload,
    sourceTag}), GET /traces/sse, and /api/metadata/* serving YAML/MD from
    src/main/resources/metadata/. POST /traces calls WorkloadQueue.receive.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ObservabilityTasks.java declaring three Task<R> constants: ROUTE (RoutingDecision),
  PROCESS (SummaryResult), EVALUATE (EvalResult).
- Domain records WorkloadItem, TracingSpan, RoutingDecision, SummaryResult, EvalResult,
  WorkloadItemState.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9281 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/workload-items.jsonl with 8 canned item lines spanning
  SUMMARISE, ESCALATE, and SKIP routing categories across workload types: "policy-doc-review",
  "log-anomaly-summary", "incident-report", "audit-trail-digest".
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: M1 hotl deployer-runtime-monitoring,
  E1 eval-periodic performance-monitor. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with decisions.authority_level = autonomous,
  oversight.human_on_loop = true, data.data_classes.pii = TO_BE_COMPLETED_BY_DEPLOYER;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router-agent.md, prompts/summary-agent.md, prompts/eval-judge.md loaded as agent
  system prompts.
- README.md at the project root: title "Akka Sample: Workflow Observability (Arize/Langfuse)",
  prerequisites (None for integration tier), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs with the App UI tab using a two-column layout (left = live item list with
  status pill, routing chip, workload-type badge; right = selected item detail with span
  waterfall, routing rationale, summary output, eval score chip). Browser title exactly:
  <title>Akka Sample: Workflow Observability</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    router-agent.json — 8–12 RoutingDecision entries spanning SUMMARISE
      (routine policy docs, log summaries, standard audits), ESCALATE
      (compliance violations detected, sensitive PII references, legal
      mentions), and SKIP (empty payloads, test items, duplicates). Each
      entry has confidence and a one-sentence rationale. The mock should
      ESCALATE under ambiguity.
    summary-agent.json — 4–6 SummaryResult entries with 2–4 sentence
      summaries, keyFindings of 1–3 bullet points as a single string,
      and realistic outputTokens (80–250).
    eval-judge.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric (relevance, factual grounding, routing
      accuracy).
- A MockModelProvider.seedFor(itemId) helper makes per-item selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- TraceExporter runs as a Consumer subscribed to SpanEmitted events — not inside
  an Agent's prompt. The span is captured at the infrastructure layer.
- The TraceExporter HTTP POST failure must be non-blocking; a failed export never
  fails the item's processing pipeline.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
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
