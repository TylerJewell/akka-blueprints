# SPEC — sentiment-monitor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Sentiment Monitor.
**One-line pitch:** A background worker watches Linear issue comments, scores sentiment on each new comment, tracks negative trends per thread, and dispatches Slack alerts before issues spiral.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI agents. Specifically:

- A **before-tool-call guardrail** fires before every `postSlackAlert` tool call to validate that the target channel is in the configured allowlist and that the alert payload does not exceed the permitted mention scope. Without this, a misconfigured or adversarially nudged agent could flood unrelated channels.
- An **eval-periodic drift watch** runs every 60 minutes, sampling recently scored comments against a small ground-truth label set to detect when the sentiment model's score distribution has drifted. Drift signals surface on the UI's Eval Matrix tab before they affect alert quality.

The result is a system that catches sentiment problems early and also watches itself for model degradation.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live thread list: every Linear issue being tracked, its most recent comment sentiment, and its current trend label (`STABLE`, `DECLINING`, `CRITICAL`).
2. `CommentPoller` (TimedAction) ticks every 20 s and inserts new simulated comments into `CommentQueue`. (Drips canned comments covering a range of sentiments.)
3. For each new comment: `SentimentScoringWorkflow` calls `SentimentScoringAgent` → produces a `SentimentScore`, emits `CommentScored` on `IssueThreadEntity`.
4. After each score, `IssueThreadEntity` recalculates the thread's running average and trend label. If the trend crosses the CRITICAL threshold, the entity emits `CriticalTrendDetected`.
5. `AlertDispatchWorkflow` starts on `CriticalTrendDetected`. The before-tool-call guardrail validates the target Slack channel. If valid, `TrendAnalysisAgent` produces an alert summary and the workflow calls `postSlackAlert`.
6. The user can silence an alert in the UI — this emits `AlertSilenced` and prevents repeated dispatches for the same thread until the trend recovers.
7. `SentimentEvalRunner` (TimedAction) ticks every 60 minutes, picks up to 10 scored comments with ground-truth labels, calls `EvalJudge`, and writes a `DriftEvalRecorded` event on `SentimentEvalEntity`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CommentPoller` | `TimedAction` | Drips simulated Linear comments into `CommentQueue` every 20 s. | scheduler | `CommentQueue` |
| `CommentQueue` | `EventSourcedEntity` | Append-only audit log of `CommentReceived` events. | `CommentPoller`, `SentimentEndpoint` | `SentimentScoringConsumer` |
| `SentimentScoringConsumer` | `Consumer` | Reads `CommentReceived` events; starts a `SentimentScoringWorkflow` per comment. | `CommentQueue` events | `SentimentScoringWorkflow` |
| `SentimentScoringAgent` | `Agent` (typed, not autonomous) | Scores a comment on a −5 to +5 integer scale with a confidence level and a one-sentence reason. | invoked by `SentimentScoringWorkflow` | returns `SentimentScore` |
| `TrendAnalysisAgent` | `AutonomousAgent` | Given a thread's recent score window, produces an `AlertSummary` suitable for Slack. | invoked by `AlertDispatchWorkflow` | returns `AlertSummary` |
| `SentimentScoringWorkflow` | `Workflow` | Per-comment orchestration: score → write to entity → check trend → conditionally start `AlertDispatchWorkflow`. | `SentimentScoringConsumer` | `IssueThreadEntity`, `AlertDispatchWorkflow` |
| `AlertDispatchWorkflow` | `Workflow` | Validates Slack channel scope (guardrail hook), calls `TrendAnalysisAgent`, dispatches alert. | `IssueThreadEntity` (CriticalTrendDetected path) | Slack tool stub |
| `IssueThreadEntity` | `EventSourcedEntity` | Lifecycle per Linear issue thread: tracks all scored comments, running average, trend label. | `SentimentScoringWorkflow`, `AlertDispatchWorkflow`, `SentimentEndpoint` | `ThreadView` |
| `SentimentEvalEntity` | `EventSourcedEntity` | Accumulates drift-eval results; one entity per eval run. | `SentimentEvalRunner` | `ThreadView` (eval sub-row) |
| `ThreadView` | `View` | Read-model row per thread plus eval summary. | `IssueThreadEntity` + `SentimentEvalEntity` events | `SentimentEndpoint` |
| `SentimentEvalRunner` | `TimedAction` | Every 60 min, picks up to 10 scored comments with ground-truth labels; calls `EvalJudge`; writes `DriftEvalRecorded`. | scheduler | `SentimentEvalEntity` |
| `SentimentEndpoint` | `HttpEndpoint` | `/api/sentiment/*` — list threads, get thread, silence alert, SSE. | — | `ThreadView`, `IssueThreadEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IssueComment(String commentId, String issueId, String authorId, String body, Instant postedAt) {}

record SentimentScore(int score, String confidence, String reason) {}
// score: −5 (most negative) to +5 (most positive). confidence: "high"|"medium"|"low".

record AlertSummary(String headline, String detail, List<String> recentReasons, Instant generatedAt) {}

record AlertDecision(boolean dispatched, String channelId, Optional<String> silencedBy, Instant decidedAt) {}

record DriftEvalResult(double meanAbsoluteError, int sampleSize, String verdict, Instant evaluatedAt) {}
// verdict: "NO_DRIFT"|"MINOR_DRIFT"|"SIGNIFICANT_DRIFT"

record IssueThread(
    String issueId,
    String issueTitle,
    List<ScoredComment> scoredComments,
    double runningAverage,
    TrendLabel trendLabel,
    int consecutiveNegativeCount,
    Optional<AlertDecision> lastAlert,
    ThreadStatus status,
    Instant firstSeenAt,
    Optional<Instant> criticalAt
) {}

record ScoredComment(String commentId, IssueComment comment, SentimentScore score, Instant scoredAt) {}

enum TrendLabel { STABLE, DECLINING, CRITICAL }
enum ThreadStatus { ACTIVE, SILENCED, RESOLVED }
```

Events on `IssueThreadEntity`: `ThreadOpened`, `CommentScored`, `TrendUpdated`, `CriticalTrendDetected`, `AlertDispatched`, `AlertSilenced`, `ThreadResolved`.

Events on `CommentQueue`: `CommentReceived`.

Events on `SentimentEvalEntity`: `DriftEvalRecorded`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/sentiment/threads` — list all tracked threads. Optional `?status=…`.
- `GET /api/sentiment/threads/{issueId}` — one thread with full scored-comment history.
- `POST /api/sentiment/threads/{issueId}/silence` — body `{ silencedBy }` → emits `AlertSilenced`; sets thread status to SILENCED.
- `POST /api/sentiment/threads/{issueId}/resolve` — body `{ resolvedBy }` → emits `ThreadResolved`; sets thread status to RESOLVED.
- `GET /api/sentiment/sse` — Server-Sent Events for every thread state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata files.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Sentiment Monitor</title>`.

App UI tab: live thread list on the left, sorted by trend severity (CRITICAL first, then DECLINING, then STABLE). Selected thread detail on the right shows the last 10 scored comments with score chips, the running average sparkline (rendered as text), and a Silence/Resolve action row.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on the `postSlackAlert(channelId, body, mentionList)` tool: blocks the call unless `channelId` is in the configured allowlist and `mentionList` does not include `@channel` or `@here`. Returns a structured error with `CHANNEL_NOT_ALLOWED` or `MENTION_SCOPE_EXCEEDED` code.
- **E1 — eval-periodic drift watch** (every 60 minutes): compares `SentimentScoringAgent` scores for a sample of comments against ground-truth labels; computes mean absolute error; records `DriftEvalRecorded` with a `NO_DRIFT | MINOR_DRIFT | SIGNIFICANT_DRIFT` verdict.

## 9. Agent prompts

- `SentimentScoringAgent` → `prompts/sentiment-scorer.md`. Typed scorer. Always returns a `SentimentScore` with an integer score in [−5, +5].
- `TrendAnalysisAgent` → `prompts/trend-analyst.md`. AutonomousAgent. Given a thread's recent score window, produces an `AlertSummary`.
- `EvalJudge` (used by `SentimentEvalRunner`) → `prompts/eval-judge.md`. Compares predicted scores to ground-truth labels; reports drift verdict.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a comment; it appears in the UI within 20 s; its sentiment score and updated thread trend are visible.
2. **J2** — Five consecutive negative comments on the same thread tip it to CRITICAL → an alert is dispatched to the allowed Slack channel (simulated).
3. **J3** — Guardrail rejects a dispatch to a channel not in the allowlist; the workflow records the rejection without crashing.
4. **J4** — User silences an alert; no further dispatches occur until the thread is resolved.
5. **J5** — Eval runner runs and surfaces a drift verdict on the Eval Matrix tab.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sentiment-monitor demonstrating the continuous-monitor × cx-support
cell. Runs out of the box (in-memory Linear feed simulator; no real Linear or Slack
integration). Maven group io.akka.samples. Artifact id
continuous-monitor-cx-support-sentiment-monitor. Java package
io.akka.samples.sentimentanalysistrackingonlinearissuestoslack. HTTP port 9155.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) SentimentScoringAgent — typed scorer. System prompt
  loaded from prompts/sentiment-scorer.md. Input: IssueComment{commentId, issueId,
  authorId, body, postedAt}. Output: SentimentScore{score: int (−5 to +5), confidence:
  "high"|"medium"|"low", reason: String}.

- 1 AutonomousAgent TrendAnalysisAgent — definition() with capability(TaskAcceptance
  .of(ALERT).maxIterationsPerTask(3)). System prompt from prompts/trend-analyst.md.
  Input: TrendWindow{issueId, issueTitle, recentScores: List<SentimentScore>,
  runningAverage: double, consecutiveNegativeCount: int}. Output: AlertSummary{headline,
  detail, recentReasons: List<String>, generatedAt}.

- 1 AutonomousAgent EvalJudge — definition() with capability(TaskAcceptance.of(DRIFT_EVAL)
  .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md. Input:
  List<LabeledScore{commentId, predictedScore, groundTruthScore}>. Output:
  DriftEvalResult{meanAbsoluteError: double, sampleSize: int, verdict: String
  ("NO_DRIFT"|"MINOR_DRIFT"|"SIGNIFICANT_DRIFT"), evaluatedAt: Instant}.

- 2 Workflows:
  * SentimentScoringWorkflow — per comment. Steps: scoreStep -> writeTrendStep ->
    conditionalAlertStep. scoreStep wraps SentimentScoringAgent with stepTimeout 15s.
    writeTrendStep calls IssueThreadEntity.recordScore; if entity returns
    CriticalTrendDetected, conditionalAlertStep starts AlertDispatchWorkflow. On timeout
    in scoreStep, emit CommentScored with score=0 and confidence="low" (neutral fallback).
  * AlertDispatchWorkflow — per critical detection. Steps: validateChannelStep ->
    summariseStep -> dispatchStep. validateChannelStep is the guardrail hook: reads the
    configured channel allowlist from application.conf; rejects with CHANNEL_NOT_ALLOWED
    if not found or MENTION_SCOPE_EXCEEDED if mentionList contains @channel or @here.
    summariseStep calls TrendAnalysisAgent with a TrendWindow (timeout 30s). dispatchStep
    calls the postSlackAlert(channelId, body, mentionList) tool stub; records result in
    IssueThreadEntity.recordAlertDispatched.

- 3 EventSourcedEntities:
  * CommentQueue — append-only audit log. Command receive(IssueComment) emits
    CommentReceived{comment}.
  * IssueThreadEntity (one per issueId) — full thread lifecycle. State IssueThread{issueId,
    issueTitle, scoredComments: List<ScoredComment>, runningAverage: double, trendLabel:
    TrendLabel (STABLE/DECLINING/CRITICAL), consecutiveNegativeCount: int,
    lastAlert: Optional<AlertDecision>, status: ThreadStatus (ACTIVE/SILENCED/RESOLVED),
    firstSeenAt: Instant, criticalAt: Optional<Instant>}. Events: ThreadOpened,
    CommentScored, TrendUpdated, CriticalTrendDetected, AlertDispatched, AlertSilenced,
    ThreadResolved. Commands: openThread, recordScore, recordAlertDispatched, silenceAlert,
    resolveThread, getThread. emptyState() returns IssueThread.initial("", "") without
    commandContext() reference.
  * SentimentEvalEntity (one per eval-run-id) — accumulates drift-eval result.
    Events: DriftEvalRecorded{result: DriftEvalResult}. Commands: recordEval, getEval.

- 1 Consumer SentimentScoringConsumer subscribed to CommentQueue events; for each
  CommentReceived, starts a SentimentScoringWorkflow with commentId as the workflow id.

- 1 View ThreadView with row type IssueThreadRow (mirrors IssueThread minus the full
  scoredComments list — the list is fetched on-demand via GET /api/sentiment/threads/{id}).
  ThreadView ALSO includes a separate EvalSummaryRow sourced from SentimentEvalEntity events,
  joined via a sub-query. ONE query getAllThreads SELECT * AS threads FROM thread_view.
  No WHERE filter — caller filters client-side.

- 2 TimedActions:
  * CommentPoller — every 20s, reads next line from
    src/main/resources/sample-events/comments.jsonl and calls CommentQueue.receive.
  * SentimentEvalRunner — every 60 minutes, queries ThreadView.getAllThreads, picks
    up to 10 ScoredComments that have ground-truth labels from
    src/main/resources/sample-events/ground-truth-labels.jsonl, calls EvalJudge,
    then calls SentimentEvalEntity.recordEval.

- 2 HttpEndpoints:
  * SentimentEndpoint at /api/sentiment with GET /threads, GET /threads/{issueId},
    POST /threads/{issueId}/silence (body {silencedBy}), POST /threads/{issueId}/resolve
    (body {resolvedBy}), GET /sse, and /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/. Silence writes AlertSilenced + sets status
    SILENCED; resolve writes ThreadResolved + sets status RESOLVED.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- SentimentTasks.java declaring three Task<R> constants: ALERT (AlertSummary),
  DRIFT_EVAL (DriftEvalResult). (SentimentScoringAgent is typed — no Task constant needed.)
- Domain records IssueComment, SentimentScore, TrendWindow, AlertSummary, AlertDecision,
  DriftEvalResult, ScoredComment, LabeledScore.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9155 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars. Also:
    sentiment.alert.allowed-channels = ["C_SUPPORT_GENERAL", "C_ESCALATIONS"]
    sentiment.alert.negative-threshold = 3  # consecutive negatives before CRITICAL
    sentiment.eval.runner.interval-seconds = 3600
- src/main/resources/sample-events/comments.jsonl with 12 canned comment lines covering
  a range from strongly negative (−4/−5) to neutral (0/1) to positive (+4/+5), with a
  cluster of 5–6 consecutive negatives on one issueId to trigger the alert path.
- src/main/resources/sample-events/ground-truth-labels.jsonl with 10 labeled entries
  (commentId, groundTruthScore) for use by the eval runner.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail before-tool-call
  channel-scope-enforcement, E1 eval-periodic drift-fairness-watch.
  Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with decisions.authority_level = alert-only,
  oversight.human_in_loop = false, oversight.human_on_loop = true,
  oversight.reviewer_role = support-team-lead, failure.failure_modes including
  "channel-spam-from-misconfigured-alert" and "model-drift-causing-missed-critical-threads";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/sentiment-scorer.md, prompts/trend-analyst.md, prompts/eval-judge.md loaded
  as agent system prompts.
- README.md at the project root: title "Akka Sample: Sentiment Monitor", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar with the App UI tab using a two-column
  layout (left = live thread list sorted CRITICAL→DECLINING→STABLE with trend badge and
  running-average chip; right = selected thread detail with last-10 scored comments, each
  comment showing score chip and confidence label, plus Silence/Resolve action row).
  Browser title exactly: <title>Akka Sample: Sentiment Monitor</title>. No subtitle on
  the Overview tab.

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
- Generate MockModelProvider.java with per-agent dispatch. Each branch reads
  src/main/resources/mock-responses/<agent>.json, picks one entry pseudo-randomly,
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    sentiment-scorer.json — 12–15 SentimentScore entries spanning the full
      range from −5 to +5, with matching confidence and one-sentence reason.
      Include 5–6 consecutive −3 to −5 entries for the same issueId to
      ensure the alert path is exercised.
    trend-analyst.json — 4–6 AlertSummary entries with a concise headline
      ("Negative trend on issue #[id]: 5 consecutive critical comments"),
      a 2-sentence detail, a list of 3 representative reasons, and a
      generatedAt timestamp.
    eval-judge.json — 6–8 DriftEvalResult entries spanning NO_DRIFT (MAE ≤ 0.5),
      MINOR_DRIFT (MAE 0.5–1.5), and SIGNIFICANT_DRIFT (MAE > 1.5).
- A MockModelProvider.seedFor(commentId) helper makes per-comment selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record (Lesson 1).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 4).
- emptyState() never calls commandContext() (Lesson 6).
- AutonomousAgent never silently downgraded to Agent (Lesson 7).
- SentimentScoringConsumer runs INSIDE a Consumer before any trend computation —
  not inside an Agent's prompt (Lesson 8).
- The before-tool-call guardrail on postSlackAlert must be enforced at the
  Workflow step level; the agent NEVER calls the tool directly (Lesson 9).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24 (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc). Without these,
  state names render black-on-black and arrow labels clip (Lesson 24).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them
  (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during generation
  (Lesson 25).
- No forbidden words in user-facing text (Lesson 23).
- Lessons 10, 11, 12, 13 apply to workflow step and entity command patterns.
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
