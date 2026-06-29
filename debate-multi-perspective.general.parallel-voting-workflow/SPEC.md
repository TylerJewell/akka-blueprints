# SPEC — parallel-voting-workflow

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Parallelization Workflow.
**One-line pitch:** Submit a task; a dispatcher briefs three independent voter agents — Feasibility, Risk, and Alignment — runs them in parallel against the same input, then aggregates their ballots into a final decision with per-vote detail.

## 2. What this blueprint demonstrates

The **debate-multi-perspective** coordination pattern wired with Akka's first-party primitives: a Workflow asks one dispatcher agent to brief three voter agents, runs those voters in parallel over the same task input, then asks the dispatcher to aggregate three independent ballots into a single decision. Voting aggregation is itself an inline evaluation — the blueprint demonstrates an **eval-event** control (`on-decision-eval`) that scores how well the aggregated decision reflects the individual votes, providing a continuous quality signal on the aggregation logic.

## 3. User-facing flows

The user opens the App UI tab and submits a task (title + description) via the form.

1. The system creates a `Task` record in `SUBMITTED` and starts a `VotingWorkflow`.
2. The dispatcher decomposes the task into three voter briefs: a feasibility focus, a risk focus, and an alignment focus.
3. The workflow forks: `FeasibilityVoter`, `RiskVoter`, and `AlignmentVoter` run concurrently over the same task description. Each returns a typed `Vote` with a ballot value, a confidence score, and a list of reasons.
4. The dispatcher aggregates the three votes into an `AggregatedDecision { decision, summary, votes, quorumMet }`.
5. If fewer than two voters agree on the same ballot value, the task enters `INCONCLUSIVE`; otherwise the plurality ballot wins and the task enters `DECIDED`.
6. If any voter times out after 60 seconds, the workflow short-circuits: the dispatcher aggregates from the returned votes and the task enters `PARTIAL` with a `failureReason` naming the missing voter.
7. Every five minutes, an eval sampler picks one decided task and asks an `AggregationJudge` whether the aggregated decision fairly reflects the individual ballots, attaching a 1–5 alignment score.

A `TaskSimulator` (TimedAction) drips a sample task every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskDispatcher` | `AutonomousAgent` | Decomposes the task into three voter briefs; later aggregates the three ballots into one final decision. | `VotingWorkflow` | returns typed result to workflow |
| `FeasibilityVoter` | `AutonomousAgent` | Votes on whether the proposed action is feasible given stated constraints. | `VotingWorkflow` | — |
| `RiskVoter` | `AutonomousAgent` | Votes on whether the proposed action introduces unacceptable risk. | `VotingWorkflow` | — |
| `AlignmentVoter` | `AutonomousAgent` | Votes on whether the proposed action aligns with declared objectives. | `VotingWorkflow` | — |
| `AggregationJudge` | `AutonomousAgent` | Scores how well the aggregated decision reflects the individual votes. | `EvalSampler` | — |
| `VotingWorkflow` | `Workflow` | Coordinates the parallel voter fan-out, the aggregation, and the quorum check. | `TaskEndpoint`, `TaskSimulatorConsumer` | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Holds the task's lifecycle (submitted → voting → decided / partial / inconclusive). | `VotingWorkflow`, `EvalSampler` | `TaskView` |
| `TaskQueue` | `EventSourcedEntity` | Logs each submitted task for replay/audit. | `TaskEndpoint`, `TaskSimulator` | `TaskSimulatorConsumer` |
| `TaskView` | `View` | List-of-tasks read model. | `TaskEntity` events | `TaskEndpoint` |
| `TaskSimulatorConsumer` | `Consumer` | Listens to `TaskQueue` events and starts a workflow per submission. | `TaskQueue` events | `VotingWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task every 60 s. | scheduler | `TaskQueue` |
| `EvalSampler` | `TimedAction` | Samples one decided task every 5 minutes for alignment scoring. | scheduler | `AggregationJudge`, `TaskEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `TaskView`, `TaskQueue`, `TaskEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record TaskSubmission(String title, String description, String submittedBy) {}

record VotingBrief(String feasibilityFocus, String riskFocus, String alignmentFocus) {}

record VoteReason(String weight, String text) {}              // weight: LOW | MEDIUM | HIGH
record Vote(String dimension, String ballot, int confidence,
            List<VoteReason> reasons, Instant votedAt) {}     // dimension: FEASIBILITY | RISK | ALIGNMENT; ballot: APPROVE | REJECT | ABSTAIN

record AggregatedDecision(String decision, String summary,
                          List<Vote> votes, boolean quorumMet,
                          Instant decidedAt) {}               // decision: PROCEED | HOLD | DECLINE

record AlignmentVerdict(int score, String rationale) {}       // score 1–5

record Task(
    String taskId,
    String title,
    TaskStatus status,
    Optional<String> description,
    Optional<VotingBrief> brief,
    Optional<Vote> feasibility,
    Optional<Vote> risk,
    Optional<Vote> alignment,
    Optional<AggregatedDecision> decision,
    Optional<String> failureReason,
    Optional<Integer> alignmentScore,
    Optional<String> alignmentRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { SUBMITTED, VOTING, DECIDED, PARTIAL, INCONCLUSIVE }
```

The raw `description` field is persisted on `TaskEntity` after `TaskCreated`; unlike the peerreview pattern there is no sanitizer. The description is what each voter receives.

### Events (on `TaskEntity`)

`TaskCreated`, `BriefingCompleted`, `FeasibilityVoteAttached`, `RiskVoteAttached`, `AlignmentVoteAttached`, `DecisionAggregated`, `VotingPartial`, `VotingInconclusive`, `AlignmentScored`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ title, description, submittedBy? }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=SUBMITTED|VOTING|DECIDED|PARTIAL|INCONCLUSIVE`.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Parallelization Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task, live list of tasks with status pills, expand-row to see the three votes, the aggregated decision, and the alignment score.

Browser title: `<title>Akka Sample: Parallelization Workflow</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid state-diagram label colours and edge-label `overflow:visible` per Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event aggregation judge** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one decided or partial task every 5 minutes and asks `AggregationJudge` whether the aggregated decision fairly reflects the individual ballots, emitting an `AlignmentScored` event with a 1–5 score and a short rationale. This is the single governance control; voting aggregation is itself the inline evaluation step.

## 9. Agent prompts

- `TaskDispatcher` → `prompts/task-dispatcher.md`. Decomposes the task into three voter briefs; later aggregates the three ballots into one final decision.
- `FeasibilityVoter` → `prompts/feasibility-voter.md`. Returns a `Vote` on the feasibility dimension.
- `RiskVoter` → `prompts/risk-voter.md`. Returns a `Vote` on the risk dimension.
- `AlignmentVoter` → `prompts/alignment-voter.md`. Returns a `Vote` on the alignment dimension.
- `AggregationJudge` → `prompts/aggregation-judge.md`. Returns an `AlignmentVerdict` scoring how well the aggregated decision reflects the votes.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a task; task progresses SUBMITTED → VOTING → DECIDED within 60 s; the expanded view shows three voter ballots and one aggregated decision with `quorumMet: true`; UI reflects each transition via SSE.
2. **J2** — Inject a voter timeout (set one voter's step timeout to 1 s); task enters PARTIAL with the aggregated decision reconciled from the remaining votes and `failureReason` naming the missing voter.
3. **J3** — Submit a task where the three voters naturally diverge (all three return different ballots); task enters INCONCLUSIVE with `quorumMet: false` and no majority decision.
4. **J4** — Wait one eval interval after a successful decision; the task row shows an alignment score (1–5) and rationale.
5. **J5** — Without any submission, the simulator drips a task every 60 s and runs the full SUBMITTED → VOTING → DECIDED path.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named parallel-voting-workflow demonstrating the
debate-multi-perspective × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
debate-multi-perspective-general-parallel-voting-workflow.
Java package io.akka.samples.parallelizationworkflow. Akka 3.6.0.
HTTP port 9227.

Components to wire (exactly):
- 5 AutonomousAgents:
  * TaskDispatcher — definition() with capability(TaskAcceptance.of(BRIEF)
    .maxIterationsPerTask(2)) AND capability(TaskAcceptance.of(AGGREGATE)
    .maxIterationsPerTask(3)). System prompt loaded from
    prompts/task-dispatcher.md.
    Returns VotingBrief{feasibilityFocus, riskFocus, alignmentFocus} for BRIEF
    and AggregatedDecision{decision, summary, votes, quorumMet, decidedAt}
    for AGGREGATE.
  * FeasibilityVoter — capability(TaskAcceptance.of(VOTE_FEASIBILITY)
    .maxIterationsPerTask(2)). System prompt from prompts/feasibility-voter.md.
    Returns Vote{dimension="FEASIBILITY", ballot, confidence, reasons:
    List<VoteReason{weight, text}>, votedAt}.
  * RiskVoter — capability(TaskAcceptance.of(VOTE_RISK)
    .maxIterationsPerTask(2)). System prompt from prompts/risk-voter.md.
    Returns Vote{dimension="RISK", ballot, confidence, reasons, votedAt}.
  * AlignmentVoter — capability(TaskAcceptance.of(VOTE_ALIGNMENT)
    .maxIterationsPerTask(2)). System prompt from prompts/alignment-voter.md.
    Returns Vote{dimension="ALIGNMENT", ballot, confidence, reasons, votedAt}.
  * AggregationJudge — capability(TaskAcceptance.of(SCORE_ALIGNMENT)
    .maxIterationsPerTask(2)). System prompt from prompts/aggregation-judge.md.
    Returns AlignmentVerdict{score, rationale}.

- 1 Workflow VotingWorkflow with steps:
  createStep -> briefStep -> [parallel] feasibilityStep, riskStep,
  alignmentStep -> joinStep -> aggregateStep -> quorumStep -> emitStep.
  createStep calls TaskEntity.createTask emitting TaskCreated (SUBMITTED).
  briefStep calls forAutonomousAgent(TaskDispatcher.class, BRIEF) -> VotingBrief;
  emits BriefingCompleted and transitions entity to VOTING.
  feasibilityStep, riskStep, alignmentStep run in parallel (CompletionStage
  zip of three calls); each wrapped with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(60)). Each attaches its Vote via
  FeasibilityVoteAttached / RiskVoteAttached / AlignmentVoteAttached.
  On any voter timeout, transition to partialStep that calls aggregateStep
  with whichever votes returned, then ends with VotingPartial (PARTIAL) and
  failureReason naming the missing voter.
  aggregateStep calls forAutonomousAgent(TaskDispatcher.class, AGGREGATE)
  with the available votes.
  quorumStep checks whether at least two votes share the same ballot value;
  if not, emits VotingInconclusive (INCONCLUSIVE); otherwise emits
  DecisionAggregated (DECIDED).
  emitStep stores the AggregatedDecision on TaskEntity.
  Override settings() with stepTimeout(60s) on the three voter steps and
  the aggregateStep, and defaultStepRecovery(maxRetries(2).failoverTo(error)).

- 1 EventSourcedEntity TaskEntity holding state Task{taskId, title,
  TaskStatus, Optional<String> description, Optional<VotingBrief> brief,
  Optional<Vote> feasibility, Optional<Vote> risk, Optional<Vote> alignment,
  Optional<AggregatedDecision> decision, Optional<String> failureReason,
  Optional<Integer> alignmentScore, Optional<String> alignmentRationale,
  Instant createdAt, Optional<Instant> finishedAt}. TaskStatus enum:
  SUBMITTED, VOTING, DECIDED, PARTIAL, INCONCLUSIVE. Events: TaskCreated,
  BriefingCompleted, FeasibilityVoteAttached, RiskVoteAttached,
  AlignmentVoteAttached, DecisionAggregated, VotingPartial,
  VotingInconclusive, AlignmentScored. Commands: createTask, completeBriefing,
  attachFeasibility, attachRisk, attachAlignment, aggregate, partial,
  inconclusive, recordAlignment, getTask. emptyState() returns
  Task.initial("", "") with no commandContext() reference.

- 1 EventSourcedEntity TaskQueue with command enqueueTask(title, description,
  submittedBy) emitting TaskReceived{taskId, title, description, submittedBy,
  receivedAt}.

- 1 View TaskView with row type TaskRow (mirrors Task minus the heavy
  description text; keep voter ballots/confidence and the aggregated decision
  but truncate reasons to counts for the list view). Table updater consumes
  TaskEntity events. ONE query getAllTasks SELECT * AS tasks FROM task_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller
  filters client-side.

- 1 Consumer TaskSimulatorConsumer subscribed to TaskQueue events; on
  TaskReceived starts a VotingWorkflow with the taskId as the workflow id,
  passing the description in the workflow start command (not the entity).

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/task-submissions.jsonl and calls
    TaskQueue.enqueueTask.
  * EvalSampler — every 5 minutes, queries TaskView.getAllTasks, picks the
    oldest DECIDED or PARTIAL task without an alignmentScore, calls
    forAutonomousAgent(AggregationJudge.class, SCORE_ALIGNMENT) with the
    three votes and the aggregated decision, then calls
    TaskEntity.recordAlignment(score, rationale).

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/. The posts list filters by status
    client-side from getAllTasks.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 Bootstrap (service-setup) that schedules the two TimedActions on startup
  and fails fast with a clear message if the configured model-provider key
  reference does not resolve (never echoing key material).

Companion files:
- VotingTasks.java declaring six Task<R> constants: BRIEF (VotingBrief),
  VOTE_FEASIBILITY (Vote), VOTE_RISK (Vote), VOTE_ALIGNMENT (Vote),
  AGGREGATE (AggregatedDecision), SCORE_ALIGNMENT (AlignmentVerdict).
- Domain records VotingBrief, VoteReason, Vote, AggregatedDecision,
  AlignmentVerdict, TaskSubmission.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9227 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-submissions.jsonl with 8 canned
  task lines covering a variety of general-purpose decision scenarios.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control (E1 eval-event
  on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/task-dispatcher.md, feasibility-voter.md, risk-voter.md,
  alignment-voter.md, aggregation-judge.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Parallelization Workflow",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live
  list with status pills; expand shows three voter ballots, aggregated decision,
  alignment score). Browser title exactly:
  <title>Akka Sample: Parallelization Workflow</title>. No subtitle on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: task-dispatcher.json, feasibility-voter.json,
  risk-voter.json, alignment-voter.json, aggregation-judge.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    task-dispatcher.json — list of either VotingBrief or AggregatedDecision
      objects. 4–6 VotingBrief entries (feasibilityFocus / riskFocus /
      alignmentFocus triples across plausible general decision tasks) and 4–6
      AggregatedDecision entries (each with a decision in {PROCEED, HOLD,
      DECLINE}, a 60–100 word summary, three nested Vote objects, quorumMet
      = true).
    feasibility-voter.json — 4–6 Vote entries with dimension="FEASIBILITY",
      a ballot in {APPROVE, REJECT, ABSTAIN}, a confidence 1–5, and 2–4
      VoteReason objects whose weight is drawn from {LOW, MEDIUM, HIGH}.
    risk-voter.json — 4–6 Vote entries with dimension="RISK".
    alignment-voter.json — 4–6 Vote entries with dimension="ALIGNMENT", at
      least one reason referencing an objective or constraint.
    aggregation-judge.json — 4–6 AlignmentVerdict entries (score 1–5 + one-
      sentence rationale on whether the aggregated decision reflects the votes).
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic
  per task id so the same task in dev produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent never silently downgraded to Agent; the extends
  clause matches the spec verbatim.
- (Lesson 4) Workflow step timeouts set explicitly on every agent-calling step
  (60s on the three voter steps and the aggregate step).
- (Lesson 6) Optional<T> for every nullable lifecycle field on the Task record
  and the View row record.
- (Lesson 7) AutonomousAgent requires the companion VotingTasks.java declaring
  every Task<R> constant.
- (Lesson 8) Verify model names against the provider's current lineup before
  locking them in application.conf.
- (Lesson 9) Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- (Lesson 10) Port 9227 declared explicitly in application.conf.
- (Lesson 11) Source-platform metadata is corpus-internal — never user-facing.
- (Lesson 12) UI fits the 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier labels are descriptive ("Runs out of the box"),
  never T1/T2/T3/T4 and never the word "deferred".
- (Lesson 23) No competitor brand names anywhere in user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc). Without these,
  state names render black-on-black and arrow labels clip.
- (Lesson 25) Never write the model-provider key value to disk; record only
  the reference.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. Exactly five .tab-panel sections; no display:none zombie
  panels — delete removed tabs from the DOM.
- Parallel voter steps use CompletionStage zip, NOT sequential calls.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, use, use, deferred.
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
