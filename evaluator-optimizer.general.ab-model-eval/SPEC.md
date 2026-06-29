# SPEC — ab-model-eval

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** A/B Testing Models.
**One-line pitch:** Submit a task prompt; two candidate agents answer it independently; a judge agent scores both against a fixed rubric, picks the winner, and logs the outcome as a typed eval event — giving operators a continuous accuracy stream for model selection decisions.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow fans out to two generator agents (`CandidateAAgent`, `CandidateBAgent`) in parallel, collects both responses, then calls an evaluator agent (`JudgeAgent`) to score them and declare a winner. The blueprint also demonstrates two governance mechanisms — an **eval-periodic** that records every trial's verdict as a typed event feeding a continuous accuracy signal, and a **ci-gate** that fires when the win-rate of the preferred model falls below a configurable recertification threshold, blocking promotion until the operator reviews.

## 3. User-facing flows

The user opens the App UI tab and submits a task prompt (a natural-language question or instruction) along with a preferred candidate label (`A` or `B`) that tracks which model the operator currently favours.

1. The system creates a `Trial` record in `RUNNING` and starts an `EvalWorkflow`.
2. The workflow fans out: `CandidateAAgent` and `CandidateBAgent` each receive the same prompt. Their calls run in parallel steps; each step has a 60-second timeout.
3. When both responses arrive, the workflow calls `JudgeAgent` with the prompt and both responses.
4. The judge scores each response across four rubric dimensions (accuracy, relevance, clarity, conciseness), returns per-dimension integers and a total, and declares a `winner` (`A`, `B`, or `TIE`).
5. The workflow emits `TrialJudged` on `TrialEntity`, transitioning the trial to `JUDGED`. The preferred candidate's win-rate is recomputed on the view side.
6. If the win-rate of the currently preferred model drops below `ab-eval.recertification.win-rate-threshold` (default 0.60) over the last `ab-eval.recertification.window-trials` trials (default 20), the workflow emits a `RecertificationRequired` flag on the entity. The App UI surfaces a warning badge.
7. If either candidate step times out, the trial transitions to `TIMED_OUT` with a structured failure reason naming which candidate failed.

A `TaskSimulator` (TimedAction) drips a canned task prompt every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CandidateAAgent` | `AutonomousAgent` | Answers a task prompt as candidate A. Persona and model governed by `prompts/candidate-a.md`. | `EvalWorkflow` | returns `CandidateResponse` to workflow |
| `CandidateBAgent` | `AutonomousAgent` | Answers the same task prompt as candidate B. Persona and model governed by `prompts/candidate-b.md`. | `EvalWorkflow` | returns `CandidateResponse` to workflow |
| `JudgeAgent` | `AutonomousAgent` | Scores both responses against the rubric; returns `JudgementRecord` with winner and per-dimension scores. | `EvalWorkflow` | returns `JudgementRecord` to workflow |
| `EvalWorkflow` | `Workflow` | Fans out to both candidates, collects responses, calls judge, evaluates recertification flag; writes events. | `EvalEndpoint`, `TaskSubmissionConsumer` | `TrialEntity` |
| `TrialEntity` | `EventSourcedEntity` | Holds trial lifecycle, both candidate responses, the judge's verdict, recertification flag, and failure reason. | `EvalWorkflow` | `TrialsView` |
| `TaskQueueEntity` | `EventSourcedEntity` | Logs each submitted task prompt for replay and audit. | `EvalEndpoint`, `TaskSimulator` | `TaskSubmissionConsumer` |
| `TrialsView` | `View` | List-of-trials read model including win-rate aggregates. | `TrialEntity` events | `EvalEndpoint` |
| `TaskSubmissionConsumer` | `Consumer` | Subscribes to `TaskQueueEntity` events; starts a workflow per submission. | `TaskQueueEntity` events | `EvalWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task prompt every 60 s from `sample-events/task-prompts.jsonl`. | scheduler | `TaskQueueEntity` |
| `AccuracySampler` | `TimedAction` | Every 30 s, scans `TrialsView`, records an `EvalRecorded` event for any judged trial that has not yet been sampled. | scheduler | `TrialEntity` |
| `EvalEndpoint` | `HttpEndpoint` | `/api/trials/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `TrialsView`, `TaskQueueEntity`, `TrialEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TaskPrompt(String text, String preferredCandidate, String submittedBy) {}

record CandidateResponse(String candidateId, String text, int tokensUsed, Instant respondedAt) {}

record RubricScores(int accuracy, int relevance, int clarity, int conciseness) {}

record JudgementRecord(
    String winner,
    RubricScores scoresA,
    RubricScores scoresB,
    int totalA,
    int totalB,
    String rationale,
    Instant judgedAt
) {}

record Trial(
    String trialId,
    String taskText,
    String preferredCandidate,
    int timeoutSeconds,
    TrialStatus status,
    Optional<CandidateResponse> responseA,
    Optional<CandidateResponse> responseB,
    Optional<JudgementRecord> judgement,
    boolean recertificationRequired,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TrialStatus { RUNNING, JUDGED, TIMED_OUT }

enum CandidateWinner { A, B, TIE }
```

### Events (on `TrialEntity`)

`TrialCreated`, `CandidateAResponded`, `CandidateBResponded`, `TrialJudged`, `TrialTimedOut`, `RecertificationFlagged`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/trials` — body `{ text, preferredCandidate?, submittedBy? }` → `{ trialId }`. Starts a workflow.
- `GET /api/trials` — list all trials. Optional `?status=RUNNING|JUDGED|TIMED_OUT`.
- `GET /api/trials/{id}` — one trial (including both responses and the full judgement).
- `GET /api/trials/sse` — server-sent events stream of every trial change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "A/B Testing Models"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, ci-gate = red).
- **App UI** — form to submit a task prompt, live list of trials with status pills, click-to-expand per-trial detail showing both responses, rubric scores, judge's rationale, and winner badge. Recertification warning badge visible when the flag is set.

Browser title: `<title>Akka Sample: A/B Testing Models</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-periodic** (`continuous-accuracy`): every completed trial's judgement is recorded as an `EvalRecorded` event with `{ winner, totalA, totalB, preferredCandidateWon, recertificationRequired }`. The `AccuracySampler` TimedAction is the canonical writer; the workflow itself also emits one on the terminal `JUDGED` transition. Enforcement: non-blocking. Events surface in the App UI's per-trial detail and in `/api/trials/{id}`.
- **CG1 — ci-gate** (`periodic-recertification-gate`): the workflow evaluates the preferred candidate's win-rate over the last `ab-eval.recertification.window-trials` (default 20) trials after each `TrialJudged` event. When win-rate drops below `ab-eval.recertification.win-rate-threshold` (default 0.60), it emits `RecertificationFlagged` on `TrialEntity`. The App UI displays a warning badge; the flag is informational (non-blocking on the trial itself) but blocks the `POST /api/trials/promote` promotion endpoint until an operator calls `POST /api/trials/recertify` to acknowledge. Enforcement: build-gate (blocks the promotion action).

## 9. Agent prompts

- `CandidateAAgent` → `prompts/candidate-a.md`. Answers a task prompt as candidate A; follows the persona constraints in the prompt file.
- `CandidateBAgent` → `prompts/candidate-b.md`. Answers the same task prompt as candidate B; follows the persona constraints in the prompt file.
- `JudgeAgent` → `prompts/judge.md`. Scores both responses across the four rubric dimensions; returns `JudgementRecord` with winner and per-dimension integers. Does not edit or rewrite responses.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — normal trial** — Submit a task prompt; trial progresses `RUNNING` → `JUDGED`; both responses, the rubric scores, and the winner are visible in the expanded trial view.
2. **J2 — recertification gate** — After enough trials where the preferred candidate loses, the App UI shows the recertification warning badge and `POST /api/trials/promote` returns `409` until the operator calls `POST /api/trials/recertify`.
3. **J3 — timeout** — Submit a task prompt in test mode where one candidate is forced to time out; the trial transitions to `TIMED_OUT` with the failure reason naming the candidate.
4. **J4 — eval timeline** — The expanded view of any `JUDGED` trial shows an `EvalRecorded` event with `winner`, `totalA`, `totalB`, and `preferredCandidateWon` populated.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ab-model-eval demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-ab-model-eval.
Java package io.akka.samples.abtestingmodels. Akka 3.6.0. HTTP port 9677.

Components to wire (exactly):
- 3 AutonomousAgents:
  * CandidateAAgent — definition() with
    capability(TaskAcceptance.of(ANSWER_A).maxIterationsPerTask(2)).
    System prompt loaded from prompts/candidate-a.md. Returns
    CandidateResponse{candidateId="A", text, tokensUsed, respondedAt}.
  * CandidateBAgent — definition() with
    capability(TaskAcceptance.of(ANSWER_B).maxIterationsPerTask(2)).
    System prompt loaded from prompts/candidate-b.md. Returns
    CandidateResponse{candidateId="B", text, tokensUsed, respondedAt}.
  * JudgeAgent — definition() with
    capability(TaskAcceptance.of(JUDGE).maxIterationsPerTask(2)).
    System prompt from prompts/judge.md. Returns
    JudgementRecord{winner, scoresA:RubricScores, scoresB:RubricScores,
    totalA, totalB, rationale, judgedAt}.

- 1 Workflow EvalWorkflow with steps:
    startStep -> [fanOutStep: candidateAStep || candidateBStep] ->
    judgeStep -> recertStep -> [recertFlag? flagStep : END] -> END.
  candidateAStep calls forAutonomousAgent(CandidateAAgent.class, trialId)
    .runSingleTask(ANSWER_A). Override stepTimeout(60s).
  candidateBStep calls forAutonomousAgent(CandidateBAgent.class, trialId)
    .runSingleTask(ANSWER_B). Override stepTimeout(60s).
  judgeStep calls forAutonomousAgent(JudgeAgent.class, trialId)
    .runSingleTask(JUDGE) passing both CandidateResponse objects.
    Override stepTimeout(60s).
  recertStep is a pure-function step: reads TrialsView win-rate for the
    preferred candidate over the last window-trials trials. If
    win-rate < win-rate-threshold, emits RecertificationFlagged; otherwise
    a no-op. The step is always non-blocking (trial still transitions to
    JUDGED regardless).
  defaultStepRecovery(maxRetries(1).failoverTo(timeoutStep)). On failover,
    timeoutStep emits TrialTimedOut with a structured failureReason naming
    the step that failed.

- 1 EventSourcedEntity TrialEntity holding state Trial{trialId, taskText,
  preferredCandidate, timeoutSeconds, TrialStatus status,
  Optional<CandidateResponse> responseA, Optional<CandidateResponse> responseB,
  Optional<JudgementRecord> judgement, boolean recertificationRequired,
  Optional<String> failureReason, Instant createdAt,
  Optional<Instant> finishedAt}.
  TrialStatus enum: RUNNING, JUDGED, TIMED_OUT.
  CandidateWinner enum: A, B, TIE.
  Events: TrialCreated, CandidateAResponded, CandidateBResponded, TrialJudged,
  TrialTimedOut, RecertificationFlagged, EvalRecorded.
  Commands: createTrial, recordResponseA, recordResponseB, recordJudgement,
  recordTimeout, flagRecertification, recordEval, getTrial.
  emptyState() returns Trial.initial("", "", "A", 60) with no commandContext()
  reference. Event-applier wraps lifecycle Optional fields with Optional.of(...).

- 1 EventSourcedEntity TaskQueueEntity with command enqueueTask(text,
  preferredCandidate, submittedBy) emitting TaskSubmitted{trialId, text,
  preferredCandidate, submittedBy, submittedAt}.

- 1 View TrialsView with row type TrialRow (mirrors Trial; also carries
  computed fields winRateA and winRateB as doubles computed over all
  JUDGED trials in the view). Table updater consumes TrialEntity events.
  ONE query getAllTrials SELECT * AS trials FROM trials_view. No WHERE
  status filter — caller filters client-side (Lesson 2).

- 1 Consumer TaskSubmissionConsumer subscribed to TaskQueueEntity events;
  on TaskSubmitted starts an EvalWorkflow with the trialId as the
  workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/task-prompts.jsonl and calls
    TaskQueueEntity.enqueueTask.
  * AccuracySampler — every 30s, queries TrialsView.getAllTrials, finds
    JUDGED trials without a matching EvalRecorded event, and calls
    TrialEntity.recordEval(winner, totalA, totalB, preferredCandidateWon,
    recertificationRequired). Idempotent per trialId.

- 2 HttpEndpoints:
  * EvalEndpoint at /api with POST /trials, GET /trials, GET /trials/{id},
    GET /trials/sse, POST /trials/promote, POST /trials/recertify, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /trials body is {text,
    preferredCandidate?, submittedBy?}; missing preferredCandidate defaults
    to "A", missing submittedBy defaults to "anonymous".
    POST /trials/promote returns 409 when recertificationRequired is true
    on any trial in the last window-trials trials.
    POST /trials/recertify clears the flag by calling
    TrialEntity.flagRecertification(false) on all flagged trials.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- EvalTasks.java declaring three Task<R> constants: ANSWER_A (resultConformsTo
  CandidateResponse), ANSWER_B (CandidateResponse), JUDGE (JudgementRecord).
- Domain records CandidateResponse, RubricScores, JudgementRecord, TaskPrompt,
  Trial; enums TrialStatus, CandidateWinner.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9677 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  ab-eval.recertification.win-rate-threshold = 0.60 and
  ab-eval.recertification.window-trials = 20, overridable by env var.
- src/main/resources/sample-events/task-prompts.jsonl with 8 canned task
  prompt lines, each shaped {"text":"...", "preferredCandidate":"A"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-periodic
  continuous-accuracy, CG1 ci-gate periodic-recertification-gate) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = model-quality-evaluation,
  decisions.authority_level = advisory, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/candidate-a.md, prompts/candidate-b.md, prompts/judge.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: A/B Testing Models",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-trial detail). Browser title exactly:
  <title>Akka Sample: A/B Testing Models</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: candidate-a.json, candidate-b.json, judge.json),
  picks one entry pseudo-randomly per call, and deserialises it into the
  agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    candidate-a.json — 6 CandidateResponse entries. Each has
      candidateId="A", a 100-200 character answer on topics in
      task-prompts.jsonl, tokensUsed between 40-80, and a respondedAt
      timestamp. Three are high-quality answers scoring well on the judge
      rubric; three are lower-quality to exercise tie and B-wins paths.
    candidate-b.json — 6 CandidateResponse entries with candidateId="B".
      Mirror structure of candidate-a.json; rotated so that the mock
      judge sees variety across the A-vs-B matchups.
    judge.json — 6 JudgementRecord entries. Two return winner="A" with
      totalA > totalB. Two return winner="B" with totalB > totalA. Two
      return winner="TIE" with equal totals. Each carries RubricScores
      for both candidates and a one-sentence rationale.
- A MockModelProvider.seedFor(trialId) helper makes the selection
  deterministic per trialId so the same trial in dev produces the same
  verdict across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. CandidateAAgent,
  CandidateBAgent, and JudgeAgent all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with an EvalTasks
  companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Trial row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: EvalTasks.java is mandatory; generating any agent without it
  is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9677, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
