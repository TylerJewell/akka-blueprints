# SPEC — hvac-analytics

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** HvacAnalytics.
**One-line pitch:** A building-operations user submits a natural-language analytical question about HVAC telemetry; one AI agent reads a current telemetry snapshot (passed as a task attachment) and returns a structured answer — a plain-language assessment, a list of supporting data points, a trend classification, and a recommended action.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `HvacAnalyticsAgent` (AutonomousAgent) carries the entire analytical decision; the surrounding components prepare its telemetry input and evaluate its output over time. One governance mechanism is wired around the agent:

- A **periodic performance evaluator** runs after each `AnswerRecorded` event, scoring the answer for analytical quality (does every supporting data point cite a real telemetry metric? is the trend classification consistent with the numbers? is the recommended action specific enough to act on?). Scores accumulate across queries so operators can track analytical accuracy over time and detect drift.

The blueprint shows that a single-agent pattern can carry meaningful ops automation without multi-agent coordination — one focused LLM call with surrounding lifecycle management and evaluation is often sufficient for targeted analytical workloads.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **zone** from a dropdown (Zone-A, Zone-B, Zone-C, or custom) and types or pastes a natural-language **question** (e.g., "Is Zone-A running above its cooling setpoint?" or "Which zone has the highest energy consumption this week?").
2. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `INITIATED` state. Within ~1 s it transitions to `SNAPSHOT_READY` — the telemetry snapshot assembled for this query is visible in the card detail (metric names, values, timestamps, zone labels).
4. Within ~10–30 s, the workflow's `analysisStep` completes. The card transitions to `ANALYSING` then `ANSWER_RECORDED`. The answer appears: a plain-language assessment paragraph, a supporting data-points table (metric, value, unit, timestamp, zone), a trend badge (IMPROVING / STABLE / DEGRADING / UNKNOWN), and a recommended action string.
5. Within ~1 s of the answer, the `evalStep` finishes. The card shows a **quality score** chip (1–5) plus a one-line rationale describing whether the answer's evidence is solid.
6. The user can submit another question; the live list keeps history visible. The Architecture tab's Eval Matrix section shows the cumulative score trend across all answered queries.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-question lifecycle: initiated → snapshot-ready → analysing → answer-recorded → evaluated. Source of truth. | `QueryEndpoint`, `TelemetryStore`, `QueryWorkflow` | `QueryView` |
| `TelemetryStore` | `Consumer` | Subscribes to `QueryInitiated` events; assembles the telemetry snapshot for the requested zone(s); calls `QueryEntity.attachSnapshot`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitSnapshotStep` → `analysisStep` → `evalStep`. | started by `TelemetryStore` once snapshot is attached | `HvacAnalyticsAgent`, `QueryEntity` |
| `HvacAnalyticsAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question in the task definition and the telemetry snapshot as a task attachment; returns `AnalyticsAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `AnswerQualityScorer` | supporting class | Deterministic rule-based scorer. Runs in `evalStep`. No LLM call. | invoked by `QueryWorkflow` | returns `EvalResult` |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TelemetryPoint(
    String metric,        // e.g. "supply-air-temp", "return-air-temp", "static-pressure"
    double value,
    String unit,          // e.g. "°F", "inWG", "kWh", "cfm"
    String zone,          // e.g. "Zone-A"
    Instant recordedAt
) {}

record TelemetrySnapshot(
    List<TelemetryPoint> points,
    Instant assembledAt,
    String zoneScope       // zone(s) included in this snapshot
) {}

record QueryRequest(
    String queryId,
    String question,
    String zoneScope,
    String submittedBy,
    Instant submittedAt
) {}

record DataPoint(
    String metric,
    double value,
    String unit,
    String zone,
    Instant recordedAt,
    String significance     // why this data point supports the answer
) {}

record AnalyticsAnswer(
    String assessment,          // plain-language 1–3 sentence paragraph
    List<DataPoint> dataPoints, // supporting evidence
    TrendClassification trend,
    String recommendedAction,
    Instant answeredAt
) {}
enum TrendClassification { IMPROVING, STABLE, DEGRADING, UNKNOWN }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<TelemetrySnapshot> snapshot,
    Optional<AnalyticsAnswer> answer,
    Optional<EvalResult> eval,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    INITIATED, SNAPSHOT_READY, ANALYSING, ANSWER_RECORDED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QueryInitiated`, `SnapshotAttached`, `AnalysisStarted`, `AnswerRecorded`, `EvaluationScored`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question, zoneScope, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: HVAC Analytics</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + trend badge + age) and a right pane with the selected query's detail — question text, telemetry snapshot table, assessment paragraph, supporting data-points table, trend badge, recommended action, and quality-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — periodic performance eval** (`eval-periodic`, `performance-monitor`): runs immediately after each `AnswerRecorded` event, as `evalStep` inside the workflow. A deterministic scorer (no LLM call — the eval is rule-based so the same answer always scores the same) checks that every `DataPoint.significance` is non-empty, that `recommendedAction` begins with an actionable verb, that `trend` is consistent with the numeric values in the data points (a trend of `IMPROVING` while every metric is above its threshold should lose a point), and that `dataPoints` is not empty. Emits `EvaluationScored` with a 1–5 score and a one-line rationale. Scores accumulate in `QueryView` so the UI can display a rolling average over the last N queries.

## 9. Agent prompts

- `HvacAnalyticsAgent` → `prompts/hvac-analytics-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached telemetry snapshot, answer the submitted question, and return one `DataPoint` per piece of supporting evidence with a significance note.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "Is Zone-A running above its cooling setpoint?" for Zone-A; within 30 s the answer appears with supporting data points and a trend badge.
2. **J2** — Three questions submitted in sequence accumulate three eval scores; the UI shows all three chips and the scores differ across questions (the seeded telemetry is varied enough to produce distinct scores).
3. **J3** — An answer whose `dataPoints` list is empty receives an eval score of 1 with a clear rationale; the card border highlights.
4. **J4** — The telemetry snapshot delivered to the agent contains zone labels but no raw equipment serial numbers — the `TelemetryStore` strips equipment-level identifiers before building the snapshot.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hvac-analytics demonstrating the single-agent × ops-automation cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-hvac-analytics. Java package
io.akka.samples.hvacdataanalyticsagent. Akka 3.6.0. HTTP port 9151.

Components to wire (exactly):

- 1 AutonomousAgent HvacAnalyticsAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/hvac-analytics-agent.md>) and
  .capability(TaskAcceptance.of(ANALYSE_TELEMETRY).maxIterationsPerTask(3)). The task receives
  the question text as its instruction text and the telemetry snapshot as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: AnalyticsAnswer{assessment: String, dataPoints: List<DataPoint>,
  trend: TrendClassification, recommendedAction: String, answeredAt: Instant}. There is no
  before-agent-response guardrail on this agent — the single control is the post-answer
  periodic eval (E1 in eval-matrix.yaml).

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitSnapshotStep — polls QueryEntity.getQuery every 1s; on query.snapshot().isPresent()
    advances to analysisStep. WorkflowSettings.stepTimeout 15s (store is in-process and fast).
  * analysisStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    HvacAnalyticsAgent.class, "analyst-" + queryId).runSingleTask(
      TaskDef.instructions(query.request().get().question())
        .attachment("telemetry.json",
          serializeSnapshotToJson(query.snapshot().get()).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANALYSE_TELEMETRY) to fetch the answer.
    On success calls QueryEntity.recordAnswer(answer). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).
  * evalStep — runs a deterministic rule-based AnswerQualityScorer (NOT an LLM call) over the
    recorded answer: checks that every DataPoint.significance is non-empty, that
    recommendedAction begins with an actionable verb, that dataPoints is non-empty, and that
    trend classification is plausible given the numeric values in dataPoints. Emits
    EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, snapshot: Optional<TelemetrySnapshot>,
  answer: Optional<AnalyticsAnswer>, eval: Optional<EvalResult>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: INITIATED,
  SNAPSHOT_READY, ANALYSING, ANSWER_RECORDED, EVALUATED, FAILED. Events: QueryInitiated{request},
  SnapshotAttached{snapshot}, AnalysisStarted{}, AnswerRecorded{answer},
  EvaluationScored{eval}, QueryFailed{reason}. Commands: initiate, attachSnapshot,
  markAnalysing, recordAnswer, recordEvaluation, fail, getQuery. emptyState() returns
  Query.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer TelemetryStore subscribed to QueryEntity events; on QueryInitiated assembles a
  TelemetrySnapshot by reading from the in-process seeded telemetry data store (a static map
  populated at startup from src/main/resources/sample-events/telemetry-data.jsonl), filtering
  to points matching the requested zoneScope, stripping any equipment-level serial-number
  fields (only zone labels are included in the snapshot), then calls
  QueryEntity.attachSnapshot(snapshot). After attachSnapshot lands, the same Consumer starts
  a QueryWorkflow with id = "query-" + queryId.

- 1 View QueryView with row type QueryRow (mirrors Query). Table updater consumes QueryEntity
  events. ONE query getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter
  — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question, zoneScope, submittedBy};
    mints queryId; calls QueryEntity.initiate; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANALYSE_TELEMETRY = Task.name("Analyse
  telemetry").description("Read the attached telemetry snapshot and produce an AnalyticsAnswer
  for the submitted question").resultConformsTo(AnalyticsAnswer.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TelemetryPoint, TelemetrySnapshot, QueryRequest, DataPoint, AnalyticsAnswer,
  TrendClassification, EvalResult, Query, QueryStatus.

- AnswerQualityScorer.java — pure deterministic logic (no LLM). Inputs: AnalyticsAnswer.
  Outputs: EvalResult. Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9151 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The HvacAnalyticsAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/telemetry-data.jsonl with seeded telemetry for three zones
  (Zone-A, Zone-B, Zone-C). Each zone has 10 data points covering: supply-air-temp,
  return-air-temp, mixed-air-temp, static-pressure, airflow-cfm, cooling-energy-kwh,
  heating-energy-kwh, outdoor-air-temp, zone-humidity, equipment-runtime-hours. Values are
  realistic for a commercial building running slightly above cooling setpoint in Zone-A,
  normally in Zone-B, and with a degrading trend in Zone-C (rising return-air-temp over
  the last 3 recorded hours).

- src/main/resources/sample-events/sample-questions.jsonl with 6 seeded question examples:
  "Is Zone-A running above its cooling setpoint?", "Which zone has the highest energy
  consumption this week?", "Is Zone-C showing signs of a degrading trend?",
  "What is the current airflow in Zone-B and is it within spec?",
  "Compare Zone-A and Zone-B cooling energy consumption.",
  "Flag any zones where return-air-temp exceeds supply-air-temp by more than 5°F."

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching the mechanism in Section 8
  of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with sector = building-operations,
  decisions.authority_level = recommend-only (the agent's answer is advisory),
  oversight.human_in_loop = true (a building engineer reads the answer before acting),
  failure.failure_modes including "inaccurate-trend-classification",
  "missing-data-points", "non-actionable-recommendation", "stale-telemetry-response";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/hvac-analytics-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: HVAC Data Analytics Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with question, snapshot table,
  assessment, data-points table, trend badge, recommended action, quality-score chip).
  Browser title exactly: <title>Akka Sample: HVAC Analytics</title>. No subtitle on the
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    analyse-telemetry.json — 8 AnalyticsAnswer entries covering all four TrendClassification
      values and three distinct zones. Each entry has a 2-sentence assessment, a dataPoints
      list of 3–5 entries (each with non-empty significance), and a recommendedAction
      beginning with an actionable verb. Plus 2 deliberately thin entries (one with an empty
      dataPoints list, one with a recommendedAction that begins with a non-actionable phrase
      like "The system appears to be") — these exercise the low-score eval path (J3). The
      mock should select a thin entry on the FIRST iteration of every 4th query (modulo seed).
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. HvacAnalyticsAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (analysisStep 60s,
  awaitSnapshotStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>.
- Lesson 7: QueryTasks.java with ANALYSE_TELEMETRY = Task.name(...).description(...)
  .resultConformsTo(AnalyticsAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9151 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (HvacAnalyticsAgent).
  The on-answer eval is rule-based (AnswerQualityScorer.java) and does NOT make an LLM
  call — keeping the pattern's "one agent" promise honest.
- The telemetry snapshot is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated analysisStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
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
