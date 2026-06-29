# SPEC — traced-agent-otel

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** TracedAgent.
**One-line pitch:** A user submits a prompt; one AI agent answers it while emitting an OpenTelemetry span for every LLM turn, tool call, and memory operation, then exports the complete span tree — including the wrapping Akka workflow spans — to a Zipkin-compatible backend for end-to-end distributed tracing.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain with full observability instrumentation wired around one `TracedConversationAgent` (AutonomousAgent). Two governance mechanisms are wired around the agent:

- A **periodic performance monitor** fires after each agent run, reading the exported span data to compute p50/p95 latency buckets, LLM-turn count, tool-call success rate, and a composite quality score. It produces a `PerformanceMonitorResult` for the entity so the UI can surface degradation trends.
- A **deployer-runtime monitor** subscribes to span-export lifecycle events and records export health — whether the OTLP exporter is reachable, how many spans were flushed vs. dropped, and the timestamp of the last successful export. This is the primary data channel for a deployer's monitoring infrastructure to observe the agent at runtime.

The blueprint shows that tracing is not just a debugging tool: the exported span stream feeds both governance mechanisms, turning observability data into measurable accountability signals.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a prompt into the **Prompt** textarea (or picks one of three seeded examples — a code-generation request, a structured-data extraction task, a multi-step reasoning problem).
2. The user optionally enables **tool mode** (a toggle) to include a web-search stub tool call in the run, so tool spans appear in the trace.
3. The user clicks **Run agent**. The UI POSTs to `/api/conversations` and receives a `conversationId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `RUNNING` — the workflow has started the agent run.
5. Within ~10–30 s the agent finishes. The card transitions to `COMPLETED`. The right pane shows the agent's final answer and the span summary: total span count, LLM-turn count, tool-call count, and aggregate wall-clock time.
6. Within ~1 s, the `spanFlushStep` completes. The card shows a **trace link** (a URL to the Zipkin UI for the root trace ID) and the span-flush result (spans exported, spans dropped).
7. Within ~2 s, the `monitorStep` fires. The card shows a **performance chip** with the p50 latency, the quality score (1–5), and a one-line monitor rationale.
8. The user can submit another prompt; the live list keeps all history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationEndpoint` | `HttpEndpoint` | `/api/conversations/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle: submitted → running → completed → flushed → monitored. Source of truth. | `ConversationEndpoint`, `SpanCollector`, `TraceExportWorkflow` | `ConversationView` |
| `SpanCollector` | `Consumer` | Subscribes to `AgentRunCompleted` events; aggregates per-span records into a `SpanSummary`; calls `ConversationEntity.attachSpanSummary`. | `ConversationEntity` events | `ConversationEntity` |
| `TraceExportWorkflow` | `Workflow` | One workflow per conversation. Steps: `agentStep` → `spanFlushStep` → `monitorStep`. | started by `ConversationEndpoint` after entity write | `TracedConversationAgent`, `ConversationEntity`, `PerformanceMonitor` |
| `TracedConversationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the prompt as task instructions; emits one `SpanRecord` per internal operation (LLM turn, tool call, memory read/write) alongside the final `AgentResponse`. | invoked by `TraceExportWorkflow` | returns `AgentResponse` |
| `SpanInstrumentor` | supporting class | Wraps each agent hook (before-LLM, after-LLM, before-tool, after-tool) and constructs `SpanRecord` objects with name, kind, start, end, attributes, and status. | called inside `TracedConversationAgent` | written to `AgentResponse.spans` |
| `PerformanceMonitor` | supporting class | Deterministic post-run scorer: reads the `SpanSummary`, computes p50/p95 from wall-clock spans, evaluates tool-call success rate, and returns a `PerformanceMonitorResult`. No LLM call. | invoked by `TraceExportWorkflow.monitorStep` | returns `PerformanceMonitorResult` |
| `ConversationView` | `View` | Read model: one row per conversation for the UI. | `ConversationEntity` events | `ConversationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum SpanKind { LLM_CALL, TOOL_CALL, MEMORY_READ, MEMORY_WRITE, WORKFLOW_STEP }
enum SpanStatus { OK, ERROR, UNSET }

record SpanRecord(
    String spanId,
    String parentSpanId,   // null if root
    String traceId,
    String operationName,
    SpanKind kind,
    Instant startTime,
    Instant endTime,
    Map<String, String> attributes,   // e.g. model, tool.name, token.count
    SpanStatus status,
    Optional<String> errorMessage
) {}

record SpanSummary(
    String traceId,
    int totalSpans,
    int llmTurnCount,
    int toolCallCount,
    int errorCount,
    long wallClockMillis,
    List<SpanRecord> spans,
    Instant collectedAt
) {}

record PromptRequest(
    String conversationId,
    String promptText,
    boolean toolModeEnabled,
    String submittedBy,
    Instant submittedAt
) {}

record AgentResponse(
    String answerText,
    List<SpanRecord> spans,
    Instant respondedAt
) {}

record PerformanceMonitorResult(
    int qualityScore,        // 1..5
    long p50Millis,
    long p95Millis,
    double toolSuccessRate,  // 0.0..1.0
    String rationale,
    Instant monitoredAt
) {}

record FlushResult(
    int spansExported,
    int spansDropped,
    boolean exporterHealthy,
    String zipkinTraceUrl,
    Instant flushedAt
) {}

record Conversation(
    String conversationId,
    Optional<PromptRequest> request,
    Optional<AgentResponse> agentResponse,
    Optional<SpanSummary> spanSummary,
    Optional<FlushResult> flushResult,
    Optional<PerformanceMonitorResult> monitorResult,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ConversationStatus {
    SUBMITTED, RUNNING, COMPLETED, FLUSHED, MONITORED, FAILED
}
```

Events on `ConversationEntity`: `PromptSubmitted`, `AgentRunStarted`, `AgentRunCompleted`, `SpanSummaryAttached`, `TraceFlushed`, `PerformanceMonitorScored`, `ConversationFailed`.

Every nullable lifecycle field on the `Conversation` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ promptText, toolModeEnabled, submittedBy }` → `{ conversationId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation.
- `GET /api/conversations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: TracedAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted conversations (status pill + performance chip + age) and a right pane with the selected conversation's detail — the prompt text, agent answer, span summary tree, flush result with the Zipkin trace URL, and the performance monitor chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — Deployer-runtime monitor** (`hotl`, `deployer-runtime-monitoring`): `SpanCollector` subscribes to `AgentRunCompleted` events; `TraceExportWorkflow.spanFlushStep` calls the OTLP exporter and records the result as `FlushResult`. A `TraceFlushed` event carries `spansExported`, `spansDropped`, `exporterHealthy`, and `zipkinTraceUrl`. Deployer monitoring infrastructure subscribes to these events and acts on `exporterHealthy = false` or a rising `spansDropped` counter.

- **E1 — Periodic performance monitor** (`eval-periodic`, `performance-monitor`): `PerformanceMonitor` runs inside `TraceExportWorkflow.monitorStep` immediately after each flush. Reads the `SpanSummary` from the entity: computes p50/p95 wall-clock latency across LLM-turn spans, tool-call success rate (OK spans / total tool spans), flags pathological distributions (all zero-duration spans indicate instrumentation failure). Emits `PerformanceMonitorScored` with a 1–5 quality score and a one-line rationale. No LLM call — deterministic and rule-based.

## 9. Agent prompts

- `TracedConversationAgent` → `prompts/traced-conversation-agent.md`. The single decision-making LLM. System prompt instructs it to answer the user's prompt, optionally call the provided web-search stub tool, and emit structured span metadata in its response.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seed prompt; within 30 s the agent completes, spans are exported, the performance chip appears, and the Zipkin trace URL is clickable.
2. **J2** — Tool mode enabled; the agent calls the web-search stub tool; the span tree shows at least one `TOOL_CALL` span alongside the `LLM_CALL` spans.
3. **J3** — A run where the OTLP exporter is unreachable (stub returns error): `FlushResult.exporterHealthy = false`; `spansDropped > 0`; the UI flags the card with a yellow export-warning chip; the conversation still transitions to `FLUSHED` (non-blocking).
4. **J4** — `PerformanceMonitor` receives a `SpanSummary` where all LLM-turn spans have `durationMillis = 0`; the quality score is 1 and the rationale reads "Instrumentation failure: zero-duration LLM spans detected."

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named traced-agent-otel demonstrating the single-agent × general cell.
Runs out of the box (no external services; Zipkin export stubs to an in-process no-op
exporter in local dev). Maven group io.akka.samples. Maven artifact
single-agent-general-akka-traced-agent-otel. Java package
io.akka.samples.durableagentwithopentelemetryzipkintracing. Akka 3.6.0. HTTP port 9621.

Components to wire (exactly):

- 1 AutonomousAgent TracedConversationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/traced-conversation-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_PROMPT).maxIterationsPerTask(5)). The task receives
  the user's prompt text as TaskDef.instructions(...) and optionally a TOOL_MODE attribute.
  Output: AgentResponse{answerText: String, spans: List<SpanRecord>, respondedAt: Instant}.
  SpanInstrumentor is a helper class that wraps before-LLM, after-LLM, before-tool,
  after-tool hooks; it constructs SpanRecord objects with traceId, spanId, parentSpanId,
  kind, startTime, endTime, attributes, status, errorMessage. The agent collects all
  SpanRecord objects into the AgentResponse.spans list. The agent does NOT call a
  real OpenTelemetry SDK — it builds SpanRecord POJOs; the export step handles the wire
  protocol.

- 1 Workflow TraceExportWorkflow per conversationId with three steps:
  * agentStep — emits AgentRunStarted, then calls componentClient.forAutonomousAgent(
    TracedConversationAgent.class, "agent-" + conversationId).runSingleTask(
      TaskDef.instructions(request.promptText())
        .attribute("TOOL_MODE", String.valueOf(request.toolModeEnabled()))
    ) — returns a taskId; fetches result via forTask(taskId).result(ANSWER_PROMPT).
    On success calls ConversationEntity.recordAgentResponse(response).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(TraceExportWorkflow::error).
  * spanFlushStep — reads the AgentResponse.spans list from the entity; calls
    OtlpExporter.flush(spans) (a local stub implementing ExporterPort interface);
    records FlushResult{spansExported, spansDropped, exporterHealthy, zipkinTraceUrl}.
    Calls ConversationEntity.recordFlush(flushResult). WorkflowSettings.stepTimeout 10s.
    This step is NON-BLOCKING on export failure — if the exporter returns an error,
    the workflow still advances to monitorStep and writes FlushResult with
    exporterHealthy=false.
  * monitorStep — runs PerformanceMonitor.score(spanSummary) deterministically. Emits
    PerformanceMonitorScored{qualityScore, p50Millis, p95Millis, toolSuccessRate,
    rationale}. WorkflowSettings.stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId, request: Optional<PromptRequest>,
  agentResponse: Optional<AgentResponse>, spanSummary: Optional<SpanSummary>,
  flushResult: Optional<FlushResult>, monitorResult: Optional<PerformanceMonitorResult>,
  status: ConversationStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  ConversationStatus enum: SUBMITTED, RUNNING, COMPLETED, FLUSHED, MONITORED, FAILED.
  Events: PromptSubmitted{request}, AgentRunStarted{}, AgentRunCompleted{agentResponse},
  SpanSummaryAttached{spanSummary}, TraceFlushed{flushResult},
  PerformanceMonitorScored{monitorResult}, ConversationFailed{reason}.
  Commands: submit, markRunning, recordAgentResponse, attachSpanSummary, recordFlush,
  recordMonitorResult, fail, getConversation. emptyState() returns
  Conversation.initial("") with no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer SpanCollector subscribed to ConversationEntity events; on AgentRunCompleted
  reads agentResponse.spans(), computes SpanSummary{traceId, totalSpans, llmTurnCount,
  toolCallCount, errorCount, wallClockMillis, spans, collectedAt}, then calls
  ConversationEntity.attachSpanSummary(summary).

- 1 View ConversationView with row type ConversationRow (mirrors Conversation minus
  agentResponse.spans raw list — the view holds the SpanSummary aggregates for the UI;
  the raw spans are available via GET /api/conversations/{id}). Table updater consumes
  ConversationEntity events. ONE query getAllConversations: SELECT * AS conversations
  FROM conversation_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with POST /conversations (body
    {promptText, toolModeEnabled, submittedBy}; mints conversationId; calls
    ConversationEntity.submit; starts TraceExportWorkflow; returns {conversationId}),
    GET /conversations (list from getAllConversations, sorted newest-first),
    GET /conversations/{id} (one row including full spans list),
    GET /conversations/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- ConversationTasks.java declaring one Task<R> constant: ANSWER_PROMPT = Task.name("Answer
  prompt").description("Process the user's prompt, emit spans for every LLM turn and
  tool call, and return an AgentResponse with the answer text and the collected span
  records").resultConformsTo(AgentResponse.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records: SpanRecord, SpanSummary, PromptRequest, AgentResponse,
  PerformanceMonitorResult, FlushResult, Conversation, SpanKind (enum),
  SpanStatus (enum), ConversationStatus (enum).

- SpanInstrumentor.java — a pure-Java helper (not an Akka component) that the agent
  uses to construct SpanRecord objects. Accepts (kind, operationName, attributes) and
  uses System.nanoTime() for start/end. Thread-safe builder pattern.

- PerformanceMonitor.java — deterministic scorer (no LLM). Inputs: SpanSummary.
  Outputs: PerformanceMonitorResult. Computes p50/p95 across LLM_CALL span durations,
  tool-call success rate, checks for zero-duration spans (instrumentation failure),
  checks that llmTurnCount > 0 (otherwise score = 1 "no LLM spans recorded").
  Scoring rubric documented in Javadoc on the class.

- OtlpExporter.java implementing ExporterPort interface. Local-dev implementation
  is an in-process no-op that logs exported span count and returns a FlushResult with
  exporterHealthy=true, spansDropped=0, and a stub zipkinTraceUrl. A
  FailingOtlpExporter test implementation returns exporterHealthy=false for use in J3.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9621 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  TracedConversationAgent.definition() binds the configured provider via the per-agent
  override pattern from the akka-context docs. Also includes:
  akka.javasdk.tracing.exporter = "otlp-stub" (default local dev stub).
  akka.javasdk.tracing.endpoint = "http://localhost:9411/api/v2/spans" (Zipkin default).

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded prompt examples:
  a code-generation request (write a function to parse ISO-8601 dates in Java),
  a structured-data extraction task (extract named entities from a given paragraph),
  a multi-step reasoning problem (a three-step arithmetic word problem).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with purpose.primary_function = observability-demo,
  decisions.authority_level = no-decisions (the agent answers prompts; it does not
  make consequential decisions on behalf of users), oversight.human_in_loop = false
  (performance monitor is automated), failure.failure_modes including
  "instrumentation-gap", "exporter-unavailable", "zero-duration-spans",
  "span-attribute-missing"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/traced-conversation-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: TracedAgent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = live list of conversation cards; right = selected-conversation detail
  with prompt text, agent answer, span summary tree, flush result + Zipkin trace URL,
  and performance chip). Browser title exactly:
  <title>Akka Sample: TracedAgent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per
  call (seedFor(conversationId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-prompt.json — 6 AgentResponse entries. Each entry has a non-empty answerText
    and a spans list containing at least 1 LLM_CALL span and (in tool-mode entries) at
    least 1 TOOL_CALL span. Span records have realistic operationNames ("llm.completion",
    "tool.web_search"), non-zero start/end times, and attributes including model name and
    token count. Plus 1 entry where all LLM_CALL spans have identical start and end times
    (zero duration) — exercises the instrumentation-failure monitor path (J4).
- MockModelProvider.seedFor(conversationId) makes per-conversation selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TracedConversationAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  ConversationTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (agentStep 90s, spanFlushStep
  10s, monitorStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Conversation row record is Optional<T>.
- Lesson 7: ConversationTasks.java with ANSWER_PROMPT is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9621 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (TracedConversationAgent).
  PerformanceMonitor and SpanCollector are non-LLM components.
- SpanInstrumentor constructs SpanRecord POJOs using System.nanoTime() — it does NOT
  depend on any external OpenTelemetry Java SDK jar. The OtlpExporter stub serialises
  the spans to JSON and could be swapped for a real OTLP HTTP exporter in production.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
