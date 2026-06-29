# SPEC — agent-metrics-monitor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Agent Metrics Monitor.
**One-line pitch:** A background worker continuously polls agent execution telemetry from BigQuery, detects performance anomalies, and surfaces health summaries for human review.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with one governance mechanism layered on top of two typed AI agents. Specifically:

- An **eval-periodic** sampler running every 30 minutes audits recent natural-language health summaries produced by `SummaryNarratorAgent`, scoring them for accuracy and grounding against the underlying metric data. This creates a continuous quality signal for the AI's narrative output.

The system's primary analytical loop — ingest metrics, detect anomalies, produce summaries — runs fully automated. The governance layer ensures that the automated summaries are independently evaluated on a schedule, surfacing drift or hallucination before it affects downstream decisions.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows a live health dashboard: one card per monitored agent, showing its current health classification, the most recent metric sample, and the latest summary excerpt.
2. `MetricsPoller` (TimedAction) ticks every 60 s and fetches the next batch of agent execution rows from the BigQuery simulator.
3. `MetricsIngestor` (Consumer) normalises rows into `AgentMetricSample` records and emits `MetricSampleRecorded` on the appropriate `AgentMetricsEntity`.
4. `MetricsScanWorkflow` starts once per batch: calls `AnomalyDetectorAgent` for each agent in the batch, then calls `SummaryNarratorAgent` to produce a window summary, then records results.
5. The health dashboard updates in real time via SSE as entity events land.
6. `EvalRunner` (TimedAction) ticks every 30 minutes, picks up to 5 recent `HealthSummary` records without an `evalScore`, calls `SummaryEvalAgent` to audit each one, and writes an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MetricsPoller` | `TimedAction` | Fetches agent metric rows from the BigQuery simulator every 60 s. | scheduler | `RawMetricsQueue` |
| `RawMetricsQueue` | `EventSourcedEntity` | Append-only audit log of `BatchReceived` events; one event per polled batch. | `MetricsPoller`, `MetricsEndpoint` | `MetricsIngestor` |
| `MetricsIngestor` | `Consumer` | Subscribes to `BatchReceived` events; normalises each row into `AgentMetricSample` and emits `MetricSampleRecorded` on the target `AgentMetricsEntity`. Starts a `MetricsScanWorkflow` per batch. | `RawMetricsQueue` events | `AgentMetricsEntity`, `MetricsScanWorkflow` |
| `AnomalyDetectorAgent` | `Agent` (typed, not autonomous) | Given a window of recent `AgentMetricSample` records for one agent, classifies its health as `HEALTHY`, `DEGRADED`, or `CRITICAL` and lists the signals driving the verdict. | invoked by Workflow | returns `AnomalyResult` |
| `SummaryNarratorAgent` | `Agent` (typed, not autonomous) | Given the full batch's `AnomalyResult` set, produces a short human-readable `HealthSummary` paragraph suitable for a dashboard. | invoked by Workflow | returns `HealthSummary` |
| `SummaryEvalAgent` | `Agent` (typed, not autonomous) | Given a `HealthSummary` and the underlying `AnomalyResult` set it was based on, scores the summary for accuracy and grounding on a 1–5 rubric. | invoked by `EvalRunner` | returns `EvalResult` |
| `MetricsScanWorkflow` | `Workflow` | Per-batch orchestration: detect anomalies per agent → narrate summary → record all results on `AgentMetricsEntity`. | `MetricsIngestor` (one workflow per `BatchReceived`) | `AgentMetricsEntity` |
| `AgentMetricsEntity` | `EventSourcedEntity` | Per-agent lifecycle: accumulates metric samples, health classifications, and summaries. State key = `agentId`. | `MetricsScanWorkflow`, `MetricsEndpoint` | `MetricsView` |
| `MetricsView` | `View` | Read-model row per agent for the UI; one query returning all agents. | `AgentMetricsEntity` events | `MetricsEndpoint` |
| `EvalRunner` | `TimedAction` | Every 30 min, picks up to 5 `HealthSummary` records without `evalScore`; calls `SummaryEvalAgent`; writes `EvalScored`. | scheduler | `AgentMetricsEntity` |
| `MetricsEndpoint` | `HttpEndpoint` | `/api/metrics/*` — list agents, get agent detail, list summaries, SSE stream. | — | `MetricsView`, `AgentMetricsEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record AgentMetricSample(
    String agentId,
    String agentName,
    Instant windowStart,
    Instant windowEnd,
    int invocationCount,
    double errorRate,       // 0.0–1.0
    double p50LatencyMs,
    double p99LatencyMs,
    int toolCallCount,
    String batchId
) {}

record AnomalyResult(
    String agentId,
    HealthStatus status,
    List<String> signals,   // e.g. ["p99LatencyMs > 8000", "errorRate > 0.15"]
    String verdict          // one-sentence summary of the classification
) {}
enum HealthStatus { HEALTHY, DEGRADED, CRITICAL }

record HealthSummary(
    String batchId,
    String narrativeText,   // 2–4 sentence paragraph
    int agentCount,
    int criticalCount,
    int degradedCount,
    Instant producedAt
) {}

record EvalResult(int score, String rationale) {}

record AgentHealthState(
    String agentId,
    String agentName,
    List<AgentMetricSample> recentSamples,   // last 10
    Optional<AnomalyResult> latestAnomaly,
    Optional<HealthSummary> latestSummary,
    Optional<Integer> latestEvalScore,
    Optional<String> latestEvalRationale,
    HealthStatus currentStatus,
    Instant firstSeenAt,
    Instant lastUpdatedAt
) {}
```

Events on `AgentMetricsEntity`: `MetricSampleRecorded`, `AnomalyDetected`, `SummaryRecorded`, `EvalScored`.

Events on `RawMetricsQueue`: `BatchReceived` (the raw batch as audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/metrics/agents` — list all monitored agents with current health state. Optional `?status=CRITICAL|DEGRADED|HEALTHY`.
- `GET /api/metrics/agents/{agentId}` — full detail for one agent including recent samples and latest summary.
- `GET /api/metrics/summaries` — list recent `HealthSummary` records, newest-first.
- `GET /api/metrics/sse` — Server-Sent Events for every agent health state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Agent Metrics Monitor</title>`.

App UI tab: a **health dashboard** with one card per monitored agent, sorted by severity (CRITICAL first). Each card shows the health status badge, agent name, key metric chips (error rate, p99 latency), and the latest summary excerpt. A right-side panel shows the selected agent's full sample history and the most recent narrative.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-periodic** (every 30 minutes): `EvalRunner` picks recent `HealthSummary` records that have not yet been scored, calls `SummaryEvalAgent` with the underlying `AnomalyResult` data, and writes `EvalScored` events carrying a 1–5 score and a one-sentence rationale. This catches summaries that overstate severity, understate risk, or introduce claims not grounded in the metric data.

## 9. Agent prompts

- `AnomalyDetectorAgent` → `prompts/anomaly-detector.md`. Typed classifier. Always returns one `AnomalyResult` per agent.
- `SummaryNarratorAgent` → `prompts/summary-narrator.md`. Produces a `HealthSummary` paragraph from the batch's full `AnomalyResult` set.
- `SummaryEvalAgent` → `prompts/summary-eval.md`. Scores a `HealthSummary` on a 1–5 rubric (accuracy, grounding, clarity).

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator batch arrives; all agent cards update within 60 s; health chips render correctly.
2. **J2** — A batch containing a high-error-rate agent produces a `CRITICAL` classification; the anomaly signals are visible in the detail panel.
3. **J3** — `SummaryNarratorAgent` produces a narrative that appears in the dashboard's summary excerpt.
4. **J4** — `EvalRunner` scores a summary within 30 minutes; the score chip appears on the summary card.
5. **J5** — A subsequent batch where all agents are healthy resets their status to `HEALTHY` in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named agent-metrics-monitor demonstrating the continuous-monitor ×
governance-risk cell. Runs out of the box (in-process BigQuery simulator; no live
GCP project required). Maven group io.akka.samples. Artifact
continuous-monitor-governance-risk-agent-metrics-monitor. Java package
io.akka.samples.agentobservabilitybq. HTTP port 9322.

Components to wire (exactly):

- 3 Agents (typed, NOT autonomous):
  * AnomalyDetectorAgent — System prompt loaded from prompts/anomaly-detector.md.
    Input: AnomalyDetectorInput{agentId: String, agentName: String,
    samples: List<AgentMetricSample>}. Output: AnomalyResult{agentId, status:
    HealthStatus (enum HEALTHY/DEGRADED/CRITICAL), signals: List<String>,
    verdict: String}. Defaults to DEGRADED under uncertainty.
  * SummaryNarratorAgent — System prompt from prompts/summary-narrator.md. Input:
    NarratorInput{batchId: String, results: List<AnomalyResult>,
    agentCount: int}. Output: HealthSummary{batchId, narrativeText: String (2–4
    sentences), agentCount, criticalCount, degradedCount, producedAt: Instant}.
  * SummaryEvalAgent — System prompt from prompts/summary-eval.md. Input:
    EvalInput{summary: HealthSummary, results: List<AnomalyResult>}. Output:
    EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow MetricsScanWorkflow per batch with steps: detectStep (fan-out: one
  AnomalyDetectorAgent call per agent in the batch, results collected) →
  narrateStep (SummaryNarratorAgent called with all AnomalyResult records) →
  recordStep (emits AnomalyDetected + SummaryRecorded on each AgentMetricsEntity).
  WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(20)) for detectStep;
  stepTimeout(Duration.ofSeconds(15)) for narrateStep. Use batchId as the
  workflow id so duplicate BatchReceived events fold into one workflow.

- 2 EventSourcedEntities:
  * RawMetricsQueue — append-only audit log of inbound batches. Command
    receiveBatch(RawMetricsBatch) emits BatchReceived{batchId, rows:
    List<AgentMetricSample>, receivedAt}.
  * AgentMetricsEntity (one per agentId) — full per-agent state. State
    AgentHealthState{agentId, agentName, recentSamples: List<AgentMetricSample>
    (capped at 10), latestAnomaly: Optional<AnomalyResult>, latestSummary:
    Optional<HealthSummary>, latestEvalScore: Optional<Integer>,
    latestEvalRationale: Optional<String>, currentStatus: HealthStatus,
    firstSeenAt: Instant, lastUpdatedAt: Instant}. Events: MetricSampleRecorded,
    AnomalyDetected, SummaryRecorded, EvalScored. Commands: recordSample,
    recordAnomaly, recordSummary, recordEval, getState. emptyState() returns
    AgentHealthState.initial("", "") without commandContext() reference.

- 1 Consumer MetricsIngestor subscribed to RawMetricsQueue events; for each
  BatchReceived, iterates rows, calls AgentMetricsEntity.recordSample per row,
  then starts a MetricsScanWorkflow with batchId as the workflow id.

- 1 View MetricsView with row type AgentHealthRow (mirrors AgentHealthState minus
  the recentSamples list — caller fetches full detail via /api/metrics/agents/{id}).
  Table updater consumes AgentMetricsEntity events. ONE query getAllAgents
  SELECT * AS agents FROM agent_health_view. No WHERE status filter — caller
  filters client-side.

- 2 TimedActions:
  * MetricsPoller — every 60s, reads the next batch line from
    src/main/resources/sample-events/metric-batches.jsonl and calls
    RawMetricsQueue.receiveBatch. Each line is a RawMetricsBatch JSON object.
  * EvalRunner — every 30 minutes, queries MetricsView.getAllAgents, picks up to
    5 agents with a latestSummary that has no latestEvalScore (oldest
    lastUpdatedAt first), calls SummaryEvalAgent with the summary and the
    latestAnomaly data, then calls AgentMetricsEntity.recordEval(score, rationale)
    per agent.

- 2 HttpEndpoints:
  * MetricsEndpoint at /api with GET /metrics/agents (optional ?status= filter),
    GET /metrics/agents/{agentId}, GET /metrics/summaries, GET /metrics/sse, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. GET /metrics/agents/{agentId} returns the full
    AgentHealthState including recentSamples and latestSummary.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- Domain records AgentMetricSample, AnomalyResult, HealthSummary, EvalResult,
  AgentHealthState, RawMetricsBatch{batchId: String, rows: List<AgentMetricSample>,
  receivedAt: Instant}.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9322
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/metric-batches.jsonl with 6 canned batch lines:
  * 2 batches where all agents are HEALTHY (low error rate, normal latency)
  * 2 batches with 1 agent DEGRADED (error rate 0.12–0.18, p99 > 4000ms)
  * 2 batches with 1 agent CRITICAL (error rate > 0.25, p99 > 9000ms)
  Each batch contains 3–5 AgentMetricSample rows covering agents:
  "agent-001" (order-processor), "agent-002" (fraud-screener),
  "agent-003" (customer-router).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control: E1 eval-periodic
  performance-monitor. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filled for governance-risk domain;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/anomaly-detector.md, prompts/summary-narrator.md, prompts/summary-eval.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Agent Metrics Monitor",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs matching the formal exemplar with the App UI tab
  using a two-column layout (left = agent health card list sorted by severity;
  right = selected agent detail with metric history table, latest anomaly signals,
  and narrative summary). Browser title exactly:
  <title>Akka Sample: Agent Metrics Monitor</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch. Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry pseudo-randomly
  per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    anomaly-detector.json — 8–12 AnomalyResult entries spanning all three
      HealthStatus values. HEALTHY entries have empty signals list. DEGRADED
      entries name 1–2 threshold-breach signals. CRITICAL entries name 2–4
      signals including errorRate and p99 breaches. Under ambiguity, default
      to DEGRADED.
    summary-narrator.json — 4–6 HealthSummary entries with narrativeText of
      2–4 sentences. Never invent agent names not present in the input.
      Counts (criticalCount, degradedCount) must match the input AnomalyResult
      set.
    summary-eval.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric axes (accuracy, grounding, clarity).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- Agents are typed (Agent), not AutonomousAgent — they are synchronous classifiers,
  not tool-using loops.
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24 (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 all apply.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
