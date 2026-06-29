# SPEC — live-eval-harness

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Live Evals.
**One-line pitch:** A continuous harness scores every agent decision in production against operator rubrics and raises drift alarms when quality degrades.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`RubricEvalAgent` and `DriftWatchAgent`). Specifically:

- An **on-decision eval** runs inside a Workflow for every (or sampled) agent decision — the rubric judge scores the output before the score is stored, giving per-decision accountability.
- A **drift-fairness watch** aggregates recent per-decision scores on a rolling window every 15 minutes; the `DriftWatchAgent` looks for distributional shifts across dimensions (quality, fairness, latency proxy) and raises a `DriftAlarm` event when thresholds are crossed.

The result is a harness where each decision has an attached score and the operator receives an alarm before quality drift becomes visible to end users.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live decision feed: every scored decision, its sanitized payload summary, its rubric scores, and the current drift status.
2. `DecisionPoller` (TimedAction) ticks every 10 s and inserts new simulated agent decisions into `DecisionQueue`. (A simulator style — drips canned decisions.)
3. For each new decision: `DecisionSanitizer` (Consumer) strips user-identifiable fields; `EvalOrchestrationWorkflow` starts per decision.
4. `RubricEvalAgent` scores the sanitized decision on the configured rubric. The result is stored on `DecisionEntity` as an `EvalScored` event.
5. If the score falls below `alarmThreshold` (default 3), `DecisionEntity` emits an `EvalAlarm` event, visible as a red chip in the UI.
6. `DriftSampler` (TimedAction) ticks every 15 minutes, queries recent scores from `EvalView`, and calls `DriftWatchAgent` with a rolling window summary.
7. `DriftWatchAgent` returns `DriftAssessment{status: DriftStatus, narrative: String}`. The harness emits `DriftOk` or `DriftAlarm` on the `DriftSnapshotEntity`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DecisionPoller` | `TimedAction` | Drips simulated decisions into `DecisionQueue` every 10 s. | scheduler | `DecisionQueue` |
| `DecisionQueue` | `EventSourcedEntity` | Append-only intake log of `DecisionReceived` events. | `DecisionPoller`, `EvalEndpoint` | `DecisionSanitizer` |
| `DecisionSanitizer` | `Consumer` | Reads `DecisionReceived` events, strips PII/user-identifiable fields, emits `DecisionSanitized` on `DecisionEntity`. | `DecisionQueue` events | `DecisionEntity` |
| `RubricEvalAgent` | `Agent` (typed, not autonomous) | Scores a sanitized decision against a rubric; returns `EvalResult`. | invoked by Workflow | returns EvalResult |
| `EvalOrchestrationWorkflow` | `Workflow` | Per-decision: sanitize-check → eval → alarm-check → done. | started by `DecisionSanitizer` | `DecisionEntity` |
| `DecisionEntity` | `EventSourcedEntity` | Per-decision lifecycle: received → sanitized → evaluated → ok/alarmed. | `EvalOrchestrationWorkflow`, `EvalEndpoint` | `EvalView` |
| `DriftWatchAgent` | `AutonomousAgent` | Analyses a score window for drift across quality, fairness, and consistency dimensions. | invoked by `DriftSampler` | returns DriftAssessment |
| `DriftSampler` | `TimedAction` | Every 15 min: queries recent `EvalView` rows, assembles a window, calls `DriftWatchAgent`. | scheduler | `DriftSnapshotEntity` |
| `DriftSnapshotEntity` | `EventSourcedEntity` | Holds the latest drift assessment and alarm history. | `DriftSampler` | `EvalView` |
| `EvalView` | `View` | Read-model row per decision for the UI. | `DecisionEntity` events, `DriftSnapshotEntity` events | `EvalEndpoint` |
| `EvalEndpoint` | `HttpEndpoint` | `/api/evals/*` — list, get, SSE, drift status. | — | `EvalView`, `DecisionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record AgentDecision(String decisionId, String agentId, String modelVersion,
                     String inputSummary, String outputSummary,
                     Optional<String> userId, Instant decidedAt) {}

record SanitizedDecision(String sanitizedInput, String sanitizedOutput,
                         String agentId, String modelVersion,
                         List<String> strippedFields, Instant decidedAt) {}

record RubricDimension(String name, int score, String justification) {}

record EvalResult(String decisionId, List<RubricDimension> dimensions,
                  int overallScore, String summary, Instant evaluatedAt) {}

record EvalAlarmInfo(String decisionId, int score, int threshold, Instant firedAt) {}

record ScoreWindow(int windowSizeDecisions, double meanScore, double stdDev,
                   Map<String, Double> meanPerDimension, Instant windowEnd) {}

record DriftAssessment(DriftStatus status, String narrative,
                       List<String> flaggedDimensions, Instant assessedAt) {}

record DecisionRecord(
    String decisionId,
    AgentDecision incoming,
    Optional<SanitizedDecision> sanitized,
    Optional<EvalResult> evalResult,
    Optional<EvalAlarmInfo> alarm,
    DecisionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DecisionStatus {
    RECEIVED, SANITIZED, EVALUATED, OK, ALARMED
}

enum DriftStatus {
    OK, WATCH, ALARM
}
```

Events on `DecisionEntity`: `DecisionReceived`, `DecisionSanitized`, `EvalScored`, `EvalAlarm`, `DecisionOk`.
Events on `DecisionQueue`: `DecisionReceived` (audit log).
Events on `DriftSnapshotEntity`: `DriftAssessed`, `DriftAlarm`, `DriftResolved`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/evals` — list all scored decisions. Optional `?status=…&agentId=…`.
- `GET /api/evals/{id}` — one decision with full rubric breakdown.
- `GET /api/evals/drift` — current drift snapshot (latest `DriftSnapshotEntity` state).
- `GET /api/evals/sse` — Server-Sent Events for every decision and drift state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Live Evals</title>`.

App UI tab is the most distinctive: it shows the **live decision feed** on the left (status chips, score badge, agent ID) and the selected decision detail on the right (rubric dimensions, drift context, alarm indicator).

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision eval** (`eval-event`, `on-decision-eval` flavor, applied in `EvalOrchestrationWorkflow`): every sanitized decision is scored by `RubricEvalAgent` before the result is stored. Score and justification are immutable once emitted.
- **E2 — drift-fairness watch** (`eval-periodic`, `drift-fairness-watch` flavor, applied in `DriftSampler` + `DriftWatchAgent`): rolling 15-minute window aggregation detects distributional shifts. `DriftAlarm` event triggers a visible alert.

## 9. Agent prompts

- `RubricEvalAgent` → `prompts/rubric-eval.md`. Typed scorer. Returns one `EvalResult` per call.
- `DriftWatchAgent` → `prompts/drift-watch.md`. Autonomous agent. Returns `DriftAssessment`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a decision; it appears in the UI within 10 s; passes sanitize → eval → EVALUATED/OK.
2. **J2** — A low-score decision triggers an `EvalAlarm`; the red chip is visible in the UI.
3. **J3** — `DriftSampler` fires; drift agent returns a `DriftAssessment`; the drift panel updates.
4. **J4** — Rubric dimensions are visible per decision in the detail pane.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named live-eval-harness demonstrating the continuous-monitor × governance-risk
cell. Runs out of the box (in-memory decision stream; no real agent integration). Maven group
io.akka.samples. Artifact id continuous-monitor-governance-risk-live-eval-harness.
Java package io.akka.samples.liveevals. HTTP port 9282.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RubricEvalAgent — scorer. System prompt loaded from
  prompts/rubric-eval.md. Input: SanitizedDecision{sanitizedInput, sanitizedOutput, agentId,
  modelVersion, strippedFields: List<String>, decidedAt} + rubricDefinition: String.
  Output: EvalResult{decisionId, dimensions: List<RubricDimension{name, score, justification}>,
  overallScore: Integer 1–5, summary: String, evaluatedAt: Instant}.
  Defaults to overallScore=1 on empty or garbled input.

- 1 AutonomousAgent DriftWatchAgent — definition() with capability(TaskAcceptance.of(DRIFT_ANALYSIS)
  .maxIterationsPerTask(3)). System prompt from prompts/drift-watch.md.
  Input: ScoreWindow{windowSizeDecisions, meanScore, stdDev,
  meanPerDimension: Map<String,Double>, windowEnd}.
  Output: DriftAssessment{status: DriftStatus (enum OK/WATCH/ALARM),
  narrative: String, flaggedDimensions: List<String>, assessedAt: Instant}.

- 1 Workflow EvalOrchestrationWorkflow per decision with steps:
  evalStep (wraps RubricEvalAgent call, stepTimeout 20 s) → alarmCheckStep (compares
  overallScore to configurable alarmThreshold, default 3; if score <= threshold emits
  EvalAlarm; otherwise emits DecisionOk) → done.
  On evalStep timeout, emit EvalScored with overallScore=1 and summary="eval-timeout".
  Every workflow uses decisionId as the workflow id.

- 3 EventSourcedEntities:
  * DecisionQueue — append-only audit log of inbound decisions. Command receive(AgentDecision)
    emits DecisionReceived{incoming}.
  * DecisionEntity (one per decisionId) — full per-decision lifecycle. State
    DecisionRecord{decisionId, incoming: AgentDecision{decisionId, agentId, modelVersion,
    inputSummary, outputSummary, Optional<String> userId, decidedAt},
    Optional<SanitizedDecision> sanitized, Optional<EvalResult> evalResult,
    Optional<EvalAlarmInfo> alarm, DecisionStatus status, Instant createdAt,
    Optional<Instant> finishedAt}. DecisionStatus enum: RECEIVED, SANITIZED, EVALUATED,
    OK, ALARMED. Events: DecisionReceived, DecisionSanitized, EvalScored, EvalAlarm,
    DecisionOk. Commands: registerDecision, attachSanitized, recordEval, recordAlarm,
    markOk, getDecision. emptyState() returns DecisionRecord.initial("", null) without
    commandContext() reference.
  * DriftSnapshotEntity (single instance id="global") — holds current drift state.
    State: DriftSnapshot{status: DriftStatus, narrative, flaggedDimensions,
    lastAssessedAt, alarmCount}. Events: DriftAssessed, DriftAlarm, DriftResolved.
    Commands: recordAssessment, getSnapshot.

- 1 Consumer DecisionSanitizer subscribed to DecisionQueue events; for each DecisionReceived,
  strips userId and any token-like fields from inputSummary and outputSummary
  (replace with "[STRIPPED]"), records strippedFields list, builds SanitizedDecision,
  calls DecisionEntity.registerDecision followed by attachSanitized. Then starts an
  EvalOrchestrationWorkflow with decisionId as the workflow id.

- 1 View EvalView with row type EvalViewRow (mirrors DecisionRecord minus raw incoming userId;
  includes optional DriftStatus from DriftSnapshotEntity). ONE query getAllDecisions
  SELECT * AS decisions FROM eval_view ORDER BY createdAt DESC. Separate query
  getLatestDrift returning DriftSnapshot. No WHERE filter on main query — caller filters
  client-side.

- 2 TimedActions:
  * DecisionPoller — every 10s, reads next line from src/main/resources/sample-events/
    agent-decisions.jsonl and calls DecisionQueue.receive.
  * DriftSampler — every 15 minutes, queries EvalView.getAllDecisions, takes the most
    recent 50 EVALUATED/OK/ALARMED records, computes ScoreWindow{windowSizeDecisions,
    meanScore, stdDev, meanPerDimension, windowEnd=Instant.now()}, calls DriftWatchAgent
    with that window, then calls DriftSnapshotEntity.recordAssessment with the resulting
    DriftAssessment. Emits DriftAlarm event if status==ALARM.

- 2 HttpEndpoints:
  * EvalEndpoint at /api with GET /evals, GET /evals/{id}, GET /evals/drift,
    GET /evals/sse, and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. SSE delivers EvalViewRow updates and DriftSnapshot changes.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- EvalTasks.java declaring two Task<R> constants: RUBRIC_EVAL (EvalResult),
  DRIFT_ANALYSIS (DriftAssessment).
- Domain records AgentDecision, SanitizedDecision, RubricDimension, EvalResult,
  EvalAlarmInfo, ScoreWindow, DriftAssessment.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9282 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/agent-decisions.jsonl with 10 canned decision lines
  covering OK (high-score), ALARMED (low-score), and WATCH-boundary scores, across at
  least 3 distinct agentId values.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: E1 eval-event on-decision-eval,
  E2 eval-periodic drift-fairness-watch. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with decisions.authority_level = automated-with-monitoring,
  oversight.human_on_loop = true, oversight.alarm_threshold_configured = true,
  failure.failure_modes including "silent-quality-degradation" and "fairness-drift-undetected";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/rubric-eval.md, prompts/drift-watch.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Live Evals", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live decision feed with score badge, status chip, agentId label; right = selected
  decision detail with rubric dimension rows + drift context panel). Browser title
  exactly: <title>Akka Sample: Live Evals</title>. No subtitle on the Overview tab.

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
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    rubric-eval.json — 8–12 EvalResult entries with overallScore 1–5 and two to
      four RubricDimension entries each (names: accuracy, consistency, fairness,
      groundedness). Each dimension has its own score 1–5 and a one-sentence
      justification. Include entries at overallScore <= 3 (alarm range) so
      alarm behaviour is exercised during a normal demo run.
    drift-watch.json — 6–8 DriftAssessment entries spanning OK, WATCH, and ALARM
      statuses. ALARM entries must include at least one flaggedDimension and a
      narrative naming the drift signal (e.g., "Fairness scores dropped 0.6 std
      dev over the last 50 decisions.").
- A MockModelProvider.seedFor(decisionId) helper makes per-decision selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- DecisionSanitizer runs INSIDE a Consumer before any LLM call — not inside an Agent's
  prompt and not after the LLM has seen the raw payload.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
