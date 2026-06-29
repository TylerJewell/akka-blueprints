# SPEC — query-planner-parallel-executor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Query Planning Workflow.
**One-line pitch:** Submit a research question; a Planner decomposes it into parallel sub-queries, multiple retrieval executors run them concurrently and stream results, and a Synthesis agent merges partial answers — looping back for follow-up sub-queries when coverage is insufficient.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern where the executor tier is parallel, not sequential. The Planner produces a `QueryPlan` — an ordered list of `SubQuery` records, each tagged with a retrieval strategy (`CORPUS`, `WEB`, `KNOWLEDGE_BASE`) and a concurrency group. The workflow fans out all sub-queries in a group simultaneously, collects their `SubQueryResult` records, then asks the Planner to evaluate coverage and decide: issue another round of sub-queries, proceed to synthesis, or fail.

The blueprint also demonstrates two governance controls wired into that loop:

- a **before-tool-call guardrail** that vets each sub-query before the retrieval executor runs, blocking queries that name disallowed hosts, request document categories outside the configured scope, or carry prompt-injection markers in the query text.
- a **plan-quality eval-event** that scores the Planner's `QueryPlan` before the first fan-out commits retrieval resources, surfacing low-quality plans as observable evaluation events and optionally blocking fan-out when the plan score falls below the configured threshold.

## 3. User-facing flows

The user opens the App UI tab and submits a research question via the form.

1. The system creates a `QuerySession` record in `PLANNING` and starts a `QueryWorkflow`.
2. The Planner reads the question and drafts a `QueryPlan { subQueries, coverageGoal, round }`. The workflow emits `SessionPlanned`.
3. **Plan-quality eval:** before fan-out, the workflow runs `PlanQualityEvaluator.score(plan)`. The result is recorded as an evaluation event. If the score is below the threshold, the workflow requests a revised plan (up to two revisions). On a third low-quality plan, the session moves to `FAILED`.
4. The workflow fans out all sub-queries in `round 1` simultaneously. Each retrieval executor runs its sub-query and returns a `SubQueryResult`.
5. The **before-tool-call guardrail** vets each sub-query before its executor receives it. On rejection the sub-query is marked `BLOCKED_BY_GUARDRAIL` and the Planner is informed before the next coverage evaluation.
6. As results arrive they are streamed via SSE. When all sub-queries in the round are complete (or blocked/failed), the Planner evaluates coverage: `SUFFICIENT`, `NEEDS_FOLLOWUP`, or `FAIL`.
7. On `NEEDS_FOLLOWUP` the Planner emits a new `QueryPlan` for the next round (up to three rounds total). On `SUFFICIENT`, the workflow calls `SynthesisAgent` which merges all results into a `ResearchAnswer { summary, citations, confidence }`.
8. The operator can press **Halt new dispatches** in the dashboard at any time. The workflow finishes the in-flight round, then ends with `SessionHaltedOperator`. The session moves to `HALTED`.

A `QuerySimulator` (TimedAction) drips a sample question every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Decomposes the question into `SubQuery` records, scores coverage after each round, decides next action. | `QueryWorkflow` | returns typed result to workflow |
| `CorpusSearchExecutor` | `AutonomousAgent` | Runs sub-queries tagged `CORPUS` against seeded document-corpus fixtures. | `QueryWorkflow` | — |
| `WebLookupExecutor` | `AutonomousAgent` | Runs sub-queries tagged `WEB` against seeded web-fixture data. | `QueryWorkflow` | — |
| `KnowledgeBaseExecutor` | `AutonomousAgent` | Runs sub-queries tagged `KNOWLEDGE_BASE` against seeded keyword-index fixtures. | `QueryWorkflow` | — |
| `SynthesisAgent` | `AutonomousAgent` | Merges all `SubQueryResult` records into a `ResearchAnswer`. | `QueryWorkflow` | returns `ResearchAnswer` to workflow |
| `QueryWorkflow` | `Workflow` | Drives the plan → eval → fan-out → collect → coverage-decide loop, follow-up branch, synthesis, and halt branches. | `QueryEndpoint`, `QueryRequestConsumer` | `QuerySessionEntity` |
| `QuerySessionEntity` | `EventSourcedEntity` | Holds session lifecycle, plan, per-round sub-query results, and final answer. | `QueryWorkflow` | `SessionView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `QueryEndpoint` (operator action) | `QueryWorkflow` (polls) |
| `QueryQueue` | `EventSourcedEntity` | Audit log of submitted queries. | `QueryEndpoint`, `QuerySimulator` | `QueryRequestConsumer` |
| `SessionView` | `View` | List-of-sessions read model for the UI. | `QuerySessionEntity` events | `QueryEndpoint` |
| `QueryRequestConsumer` | `Consumer` | Subscribes to `QueryQueue` events; starts a `QueryWorkflow` per submission. | `QueryQueue` events | `QueryWorkflow` |
| `QuerySimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/query-prompts.jsonl` and enqueues it. | scheduler | `QueryQueue` |
| `StaleSessionMonitor` | `TimedAction` | Every 30 s, marks any session stuck in `EXECUTING` past 5 minutes as `STALE`. The workflow polls this and ends with `SessionTimedOut`. | scheduler | `QuerySessionEntity` |
| `QueryEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, get, list, SSE, operator halt. | — | `SessionView`, `QueryQueue`, `QuerySessionEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record QueryRequest(String question, String requestedBy) {}

record SubQuery(
    String subQueryId,
    String queryText,
    RetrievalStrategy strategy,
    int round
) {}

record QueryPlan(
    List<SubQuery> subQueries,
    String coverageGoal,
    int round,
    Optional<String> revisionReason
) {}

record SubQueryResult(
    String subQueryId,
    RetrievalStrategy strategy,
    String queryText,
    boolean ok,
    String content,
    Optional<String> errorReason,
    ResultVerdict verdict,
    Instant retrievedAt
) {}

record PlanEvaluation(
    double score,
    boolean passing,
    String rationale,
    Instant evaluatedAt
) {}

record ResearchAnswer(
    String summary,
    List<String> citations,
    double confidence,
    int roundsCompleted,
    Instant producedAt
) {}

record QuerySession(
    String sessionId,
    String question,
    SessionStatus status,
    Optional<QueryPlan> currentPlan,
    Optional<PlanEvaluation> lastEval,
    List<SubQueryResult> results,
    Optional<ResearchAnswer> answer,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RetrievalStrategy { CORPUS, WEB, KNOWLEDGE_BASE }
enum ResultVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, TIMED_OUT }
enum SessionStatus { PLANNING, EVALUATING, EXECUTING, SYNTHESIZING, COMPLETED, FAILED, HALTED, STALE }
enum CoverageDecision { SUFFICIENT, NEEDS_FOLLOWUP, FAIL }
```

### Events (`QuerySessionEntity`)

`SessionCreated`, `SessionPlanned`, `PlanEvaluated`, `PlanRevisionRequested`, `SubQueryDispatched`, `SubQueryBlocked`, `SubQueryRecorded`, `CoverageAssessed`, `SynthesisStarted`, `SessionCompleted`, `SessionFailed`, `SessionHaltedOperator`, `SessionTimedOut`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`QueryQueue`)

`QuerySubmitted { sessionId, question, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ question, requestedBy? }` → `202 { sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=...`.
- `GET /api/sessions/{id}` — one session (full plan + results + answer).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason? }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Query Planning Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a question, operator halt/resume control, live list of sessions with status pills, expand-row to see the current plan, per-round sub-query results, and the final research answer.

Browser title: `<title>Akka Sample: Query Planning Workflow</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on all executor agents): every `SubQuery` is vetted before the retrieval executor receives it. The vetter checks (a) the retrieval strategy allow-list, (b) for `WEB` sub-queries — the host allow-list, (c) query text for prompt-injection markers (`"ignore previous instructions"`, `"<system>"`), (d) document category scope for `CORPUS` sub-queries. On rejection the sub-query is marked `BLOCKED_BY_GUARDRAIL`, a `SubQueryBlocked` event is emitted, and the Planner is informed before the next coverage evaluation.
- **E1 — plan-quality eval-event** (`eval-event`, flavor `on-decision-eval`): before each fan-out round, `PlanQualityEvaluator.score(QueryPlan)` runs synchronously. It scores the plan on three dimensions — sub-query independence, strategy diversity, and coverage completeness — producing a `PlanEvaluation { score, passing, rationale }`. The result is recorded as a `PlanEvaluated` event regardless of outcome. When `passing=false`, the workflow requests a revised plan. Two consecutive failing evaluations trigger `SessionFailed`.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Decomposes queries; evaluates coverage; decides next round or synthesis.
- `CorpusSearchExecutor` → `prompts/corpus-search.md`. Retrieves document excerpts from corpus fixtures.
- `WebLookupExecutor` → `prompts/web-lookup.md`. Answers sub-queries from web fixture data.
- `KnowledgeBaseExecutor` → `prompts/knowledge-base.md`. Queries keyword-index fixtures.
- `SynthesisAgent` → `prompts/synthesis.md`. Merges results into a cited research answer.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "What are the key regulatory requirements for deploying autonomous AI agents in the EU and US?" Session progresses `PLANNING → EVALUATING → EXECUTING → SYNTHESIZING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a plan with 4–8 sub-queries spanning all three strategies, sub-query results for each, and a `ResearchAnswer` with citations.
2. **J2** — A sub-query whose text contains a prompt-injection marker is blocked by the guardrail; the planner is informed; the session either replans or completes via a different sub-query path.
3. **J3** — A plan that scores below the quality threshold fires a `PlanEvaluated` event with `passing=false`; the planner is asked to revise; subsequent plan passes or session fails after two revisions.
4. **J4** — Operator halts new dispatches mid-session; in-flight round finishes; no further rounds dispatch; session ends in `HALTED`.
5. **J5** — A corpus fixture result containing a credential-shaped string is scrubbed before it reaches the synthesis prompt; the `ResearchAnswer` does not contain the literal credential.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named query-planner-parallel-executor demonstrating the
planner-executor × research-intel cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
planner-executor-research-intel-query-planner-parallel-executor. Java package
io.akka.samples.queryplanningworkflow. Akka 3.6.0. HTTP port 9177.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2)) and
      capability(TaskAcceptance.of(ASSESS_COVERAGE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(SYNTHESIZE).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. DECOMPOSE returns QueryPlan.
    ASSESS_COVERAGE returns CoverageDecision tagged union (Sufficient |
    NeedsFollowup(revisedPlan) | Fail(reason)).
    SYNTHESIZE returns ResearchAnswer.
  * CorpusSearchExecutor — capability(TaskAcceptance.of(CORPUS_SEARCH)
    .maxIterationsPerTask(2)). Prompt from prompts/corpus-search.md.
    Returns SubQueryResult.
  * WebLookupExecutor — capability(TaskAcceptance.of(WEB_LOOKUP)
    .maxIterationsPerTask(2)). Prompt from prompts/web-lookup.md.
    Returns SubQueryResult.
  * KnowledgeBaseExecutor — capability(TaskAcceptance.of(KB_LOOKUP)
    .maxIterationsPerTask(2)). Prompt from prompts/knowledge-base.md.
    Returns SubQueryResult.
  * SynthesisAgent — capability(TaskAcceptance.of(SYNTHESIZE_RESULTS)
    .maxIterationsPerTask(2)). Prompt from prompts/synthesis.md.
    Returns ResearchAnswer.

- 1 Workflow QueryWorkflow with steps:
  planStep -> evalStep -> [if passing] fanOutStep -> collectStep -> coverageStep
  -> [NEEDS_FOLLOWUP loops back to evalStep with revised plan | SUFFICIENT
  proceeds to synthesisStep -> completeStep | FAIL proceeds to failStep].
  checkHaltStep runs before fanOutStep on every round; on halted=true
  transitions to haltedStep.
  guardrailStep runs per sub-query before each executor call within fanOutStep.
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), evalStep ofSeconds(30), fanOutStep
    ofSeconds(180) (covers concurrent executor calls), coverageStep
    ofSeconds(45), synthesisStep ofSeconds(90).
  defaultStepRecovery(maxRetries(2).failoverTo(QueryWorkflow::error)).
  planStep calls PlannerAgent.DECOMPOSE(question).
  evalStep calls PlanQualityEvaluator.score(plan), records PlanEvaluated
    event, on passing=false requests revision up to 2 times.
  fanOutStep fans out all sub-queries for the current round in parallel via
    forAutonomousAgent(...).runSingleTask(...) calls submitted concurrently,
    joined with CompletableFuture.allOf; each call individually guarded by
    guardrailStep logic before submission.
  collectStep gathers all results, applies SecretScrubber.scrub to each
    result's content, records SubQueryRecorded events.
  coverageStep calls PlannerAgent.ASSESS_COVERAGE(plan, results).
  synthesisStep calls SynthesisAgent.SYNTHESIZE_RESULTS(question, results).
  completeStep emits SessionCompleted on QuerySessionEntity.

- 1 EventSourcedEntity QuerySessionEntity holding QuerySession state.
  emptyState() returns QuerySession.initial. Commands: createSession,
  recordPlan, recordPlanEval, recordSubQueryBlocked, recordSubQueryResult,
  recordCoverage, startSynthesis, completeSession, failSession,
  haltOperator, timeoutSession, getSession.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global".
  State SystemControl{boolean halted, Optional<String> reason,
  Optional<Instant> haltedAt}. Commands: requestHalt(reason), clearHalt,
  get. Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(sessionId,
  question, requestedBy) emitting QuerySubmitted.

- 1 View SessionView with row type SessionRow (mirror of QuerySession minus
  full results list — truncate to last 5 SubQueryResult records plus counts;
  the UI fetches the full session by id on click). Table updater consumes
  QuerySessionEntity events. ONE query getAllSessions SELECT * AS sessions
  FROM session_view. No WHERE status filter — caller filters client-side
  (Lesson 2).

- 1 Consumer QueryRequestConsumer subscribed to QueryQueue events; on
  QuerySubmitted starts a QueryWorkflow with sessionId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 90s, reads next line from
    src/main/resources/sample-events/query-prompts.jsonl and calls
    QueryQueue.enqueueQuery.
  * StaleSessionMonitor — every 30s, queries SessionView.getAllSessions,
    filters EXECUTING sessions whose createdAt is older than 5 minutes,
    calls QuerySessionEntity.timeoutSession; QueryWorkflow polls
    QuerySessionEntity.getSession in its coverageStep and exits when
    status == STALE.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /sessions, GET /sessions (filters
    client-side from getAllSessions), GET /sessions/{id}, GET /sessions/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: DECOMPOSE
  (resultConformsTo QueryPlan), ASSESS_COVERAGE (CoverageDecision),
  SYNTHESIZE (ResearchAnswer).
- ExecutorTasks.java declaring four Task<R> constants: CORPUS_SEARCH,
  WEB_LOOKUP, KB_LOOKUP (all resultConformsTo SubQueryResult) and
  SYNTHESIZE_RESULTS (resultConformsTo ResearchAnswer).
- Domain records as listed in SPEC §5, plus a CoverageDecision sealed
  interface with permits Sufficient, NeedsFollowup(revisedPlan), Fail(reason).
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token], [REDACTED:jwt],
  [REDACTED:openai-key], [REDACTED:bearer-token], [REDACTED:high-entropy].
- application/SubQueryGuardrail.java — deterministic vetter. Reject if
  strategy is not CORPUS/WEB/KNOWLEDGE_BASE; if WEB query names a host
  not on the allow-list (akka.io, doc.akka.io, github.com,
  arxiv.org, papers.ssrn.com); if CORPUS query names a document category
  not in (regulations, whitepapers, standards, case-studies); if query
  text matches prompt-injection markers
  /(?i)(ignore previous instructions|<system>|]\s*system\s*prompt)/;
  if queryText is blank.
- application/PlanQualityEvaluator.java — deterministic scorer. Computes
  a score 0.0–1.0 on three dimensions: (a) sub-query independence —
  penalise if > 50% of sub-queries share 4+ consecutive words; (b) strategy
  diversity — score 1.0 if all three strategies are used, 0.6 if two, 0.2
  if one; (c) coverage completeness — score 1.0 if subQueries.size() >= 3.
  Final score = average. passing = score >= 0.5. Produces PlanEvaluation.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9177 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/query-prompts.jsonl with 8 canned
  research questions spanning regulatory, technical, competitive, and
  historical topics.
- src/main/resources/sample-data/corpus-fixtures.jsonl — 15 canned document
  excerpts (title, category, excerpt, source). Categories: regulations,
  whitepapers, standards, case-studies.
- src/main/resources/sample-data/web-fixtures.jsonl — 10 canned web results
  (host, path, title, excerpt). Hosts within the allow-list.
- src/main/resources/sample-data/kb-fixtures.jsonl — 10 canned keyword-index
  entries (term, definition, relatedTerms). Include one entry whose definition
  contains an AKIA-shaped key fragment for the J5 sanitizer test.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, E1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance; deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/corpus-search.md, prompts/web-lookup.md,
  prompts/knowledge-base.md, prompts/synthesis.md loaded at agent startup.
- README.md at the project root as described in blueprint README section.
  NO Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs acceptable. Five tabs: Overview,
  Architecture (4 mermaid diagrams + click-to-expand component table),
  Risk Survey (7 sub-tabs), Eval Matrix (5-column table), App UI (form +
  operator controls + live session list). Browser title exactly:
  <title>Akka Sample: Query Planning Workflow</title>. No subtitle on
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — generate MockModelProvider returning random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file.
- Bootstrap.java fails fast with a clear message if the configured key
  reference does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java dispatching on agent class + Task<R> id.
  Per-agent mock-response files in src/main/resources/mock-responses/:
    planner.json — three lists by task id:
      "DECOMPOSE" → 4–6 QueryPlan entries spanning all three strategies.
      "ASSESS_COVERAGE" → entries cycling through Sufficient, NeedsFollowup,
        Fail in a plausible order.
      "SYNTHESIZE" → 4–6 ResearchAnswer entries with 80–120 word summaries
        and 3–5 citation bullets.
    corpus-search.json — 8 SubQueryResult entries, ok=true.
    web-lookup.json — 6 SubQueryResult entries, ok=true.
    knowledge-base.json — 6 SubQueryResult entries; ONE entry's content
      must contain "AKIAIOSFODNN7EXAMPLE" so J5 sanitizer test fires.
    synthesis.json — 4 ResearchAnswer entries.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Lesson 1: AutonomousAgent base class.
- Lesson 4: WorkflowSettings.stepTimeout on every agent-calling step.
- Lesson 6: Optional<T> for every nullable field on view row and entity state.
- Lesson 7: Companion PlannerTasks.java and ExecutorTasks.java.
- Lesson 8: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: /akka:build.
- Lesson 10: HTTP port 9177.
- Lesson 11: source.platform never in user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no competitor brand names.
- Lesson 24: mermaid CSS overrides + themeVariables.
- Lesson 25: API-key sourcing five-option flow.
- Lesson 26: Tab switching by data-tab/data-panel attribute; no zombie panels.
- Overview tab Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
