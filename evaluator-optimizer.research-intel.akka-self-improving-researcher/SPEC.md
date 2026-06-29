# SPEC — self-improving-deep-researcher

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Improving Deep Research Agent.
**One-line pitch:** Submit a research query; a research agent synthesizes findings; an evaluator scores quality; the loop refines until the report is sufficient or the ceiling is reached — then a prompt-rewriter agent updates the system's own memory blocks so future research runs improve.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a research agent (`ResearchAgent`) and a quality evaluator (`EvaluatorAgent`), feeding each quality critique back into the next research attempt until the report meets the rubric or the attempt ceiling is hit. After the loop ends — regardless of outcome — a third agent (`PromptRewriterAgent`) updates the persistent `PromptMemory` blocks that seed the `ResearchAgent`'s system prompt on its next session.

The blueprint also demonstrates three governance mechanisms: a **periodic-drift eval** that watches the `PromptMemory` fingerprint for uncontrolled changes, a **recertification gate** that blocks deployment when the memory has drifted beyond a configurable threshold since the last signed-off baseline, and a **human-on-the-loop** production monitor that surfaces every memory update as an observable event stream for a designated reviewer.

## 3. User-facing flows

The user opens the App UI tab and submits a research query (a topic string plus an optional depth hint and source-type preference).

1. The system creates a `Session` record in `RESEARCHING` and starts a `ResearchSessionWorkflow`.
2. `ResearchAgent` executes attempt #1: reads the current `PromptMemory` blocks and synthesizes a `ResearchReport` containing an executive summary, a list of `EvidenceSegment` records, and a `SourceManifest`.
3. `EvaluatorAgent` scores the report against a four-dimension rubric (coverage, evidence quality, synthesis coherence, source diversity) and returns either `SUFFICIENT` with a one-line rationale, or `REFINE` with a typed `QualityNotes` payload (up to four bullets).
4. On `SUFFICIENT`, the workflow transitions the session to `ACCEPTED` and calls `PromptRewriterAgent` with the winning report and the current memory.
5. On `REFINE`, the workflow records the attempt and the quality notes on the entity, then calls `ResearchAgent` again with the `QualityNotes` attached. The agent produces attempt #2, incorporating the evaluator's guidance.
6. If the loop reaches `maxAttempts` (default 3) without a `SUFFICIENT`, the workflow ends with `MAX_ATTEMPTS_REACHED`. The `PromptRewriterAgent` is still called — it sees the failure pattern and updates the memory to avoid repeating it.
7. In both terminal cases, `PromptRewriterAgent` returns a `MemoryDiff` recording which memory blocks changed, what changed, and why. The diff is stored on the entity; the `PromptMemory` store is updated atomically.

A `QuerySimulator` (TimedAction) drips a canned research query every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchAgent` | `AutonomousAgent` | Executes a research query; synthesizes findings into a `ResearchReport`; on a refine call, incorporates prior `QualityNotes`. | `ResearchSessionWorkflow` | returns `ResearchReport` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Scores a `ResearchReport` against the quality rubric; returns `SUFFICIENT` or `REFINE` with notes. | `ResearchSessionWorkflow` | returns `QualityEvaluation` to workflow |
| `PromptRewriterAgent` | `AutonomousAgent` | Reads the completed session and the current `PromptMemory`; returns a `MemoryDiff` recording which blocks to update and why. | `ResearchSessionWorkflow` | returns `MemoryDiff` to workflow |
| `ResearchSessionWorkflow` | `Workflow` | Runs the research → evaluate → refine loop; calls the rewriter at the end regardless of outcome; halts at the ceiling. | `ResearchEndpoint`, `QueryConsumer` | `SessionEntity` |
| `SessionEntity` | `EventSourcedEntity` | Holds the session lifecycle, every research attempt, every evaluation, the memory diff, and the final outcome. | `ResearchSessionWorkflow` | `SessionsView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted query for replay and audit. | `ResearchEndpoint`, `QuerySimulator` | `QueryConsumer` |
| `SessionsView` | `View` | List-of-sessions read model. | `SessionEntity` events | `ResearchEndpoint` |
| `QueryConsumer` | `Consumer` | Subscribes to `QueryQueue` events; starts a workflow per submission. | `QueryQueue` events | `ResearchSessionWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample query every 90 s from `sample-events/research-queries.jsonl`. | scheduler | `QueryQueue` |
| `DriftSampler` | `TimedAction` | Every 60 s, hashes the current `PromptMemory` and compares to the last-known fingerprint; emits `PromptDriftRecorded` on change. | scheduler | `SessionEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `SessionsView`, `QueryQueue`, `SessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ResearchQuery(String topic, String depthHint, String sourceTypePreference, String requestedBy) {}

record EvidenceSegment(String claim, String sourceRef, double confidence) {}

record SourceManifest(List<String> sourceRefs, int totalSources) {}

record ResearchReport(
    String executiveSummary,
    List<EvidenceSegment> evidence,
    SourceManifest sources,
    Instant researchedAt
) {}

record QualityNotes(List<String> bullets, String overallRationale) {}

record QualityEvaluation(
    EvaluatorVerdict verdict,
    QualityNotes notes,
    int score,
    Instant evaluatedAt
) {}

record MemoryBlock(String blockId, String blockType, String content, Instant updatedAt) {}

record PromptMemory(List<MemoryBlock> blocks, String fingerprint, Instant lastModifiedAt) {}

record MemoryBlockChange(String blockId, String changeType, String before, String after, String reason) {}

record MemoryDiff(
    List<MemoryBlockChange> changes,
    String triggerSessionId,
    EvaluatorVerdict triggerVerdict,
    Instant diffedAt
) {}

record ResearchAttempt(
    int attemptNumber,
    ResearchReport report,
    Optional<QualityEvaluation> evaluation
) {}

record Session(
    String sessionId,
    String topic,
    String depthHint,
    int maxAttempts,
    SessionStatus status,
    List<ResearchAttempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<ResearchReport> acceptedReport,
    Optional<MemoryDiff> memoryDiff,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { RESEARCHING, EVALUATING, ACCEPTED, MAX_ATTEMPTS_REACHED }

enum EvaluatorVerdict { SUFFICIENT, REFINE }
```

### Events (on `SessionEntity`)

`SessionCreated`, `AttemptResearched`, `AttemptEvaluated`, `SessionAccepted`, `SessionMaxAttemptsReached`, `MemoryUpdated`, `QualityEvalRecorded`, `PromptDriftRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ topic, depthHint?, sourceTypePreference?, requestedBy? }` → `{ sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=RESEARCHING|EVALUATING|ACCEPTED|MAX_ATTEMPTS_REACHED`.
- `GET /api/sessions/{id}` — one session (including every attempt, every evaluation, and the memory diff).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Self-Improving Deep Research Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, ci-gate = yellow, hotl = green).
- **App UI** — form to submit a research query, live list of sessions with status pills, click-to-expand per-attempt timeline showing each report summary, the evaluator's verdict and notes, and the final memory diff.

Browser title: `<title>Akka Sample: Self-Improving Deep Research Agent</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — periodic drift eval** (`eval-periodic`, `drift-fairness-watch`): the `DriftSampler` TimedAction hashes the live `PromptMemory` every 60 s and compares it against the last known fingerprint. When the hash changes, it emits `PromptDriftRecorded` on `SessionEntity` with the old and new fingerprints, the delta block count, and a timestamp. Operators see every prompt mutation in the App UI's drift timeline. Enforcement: non-blocking (observation).
- **S1 — recertification gate** (`ci-gate`, `periodic-recertification-gate`): a build-time check that reads the `PromptMemory` baseline fingerprint from `src/main/resources/governance/prompt-baseline.json`. If the live fingerprint has drifted beyond the configured `max-diff-blocks` threshold, the check fails fast with a structured message naming the changed block IDs. This is enforced as a test in the generated Maven build, so a PR that ships with undeclared prompt drift fails CI. Enforcement: build-gate.
- **H1 — deployer runtime monitor** (`hotl`, `deployer-runtime-monitoring`): every `MemoryUpdated` event is forwarded to the `ResearchEndpoint`'s SSE stream at a dedicated `memory-updates` channel. This gives a human reviewer a live view of every prompt change without requiring access to the entity store. The channel is read-only; reviewers cannot roll back memory from the UI (that is a workflow action). Enforcement: system-level.

## 9. Agent prompts

- `ResearchAgent` → `prompts/research-agent.md`. Executes a structured research synthesis on the query; on a refine call, incorporates prior `QualityNotes` and targets the identified gaps.
- `EvaluatorAgent` → `prompts/evaluator-agent.md`. Scores a `ResearchReport` against the fixed rubric; returns `SUFFICIENT` with a one-line rationale or `REFINE` with up to four short bullets.
- `PromptRewriterAgent` → `prompts/prompt-rewriter-agent.md`. Reads the session outcome and the current memory blocks; produces a `MemoryDiff` that updates strategy hints, source preferences, or synthesis guidance without altering the agent's role definition.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a query; session progresses `RESEARCHING` → `EVALUATING` → `ACCEPTED` within the attempt ceiling; the App UI shows every attempt's report summary, evaluation verdict, and the memory diff produced after acceptance.
2. **J2 — ceiling and memory update** — Submit a query whose rubric the evaluator consistently rejects (test mode); session ends in `MAX_ATTEMPTS_REACHED`; `PromptRewriterAgent` still runs and produces a memory diff recording the failure pattern; diff is visible in the App UI.
3. **J3 — drift detection** — After any session completes and memory updates, the `DriftSampler` detects the fingerprint change and surfaces a `PromptDriftRecorded` event in the drift timeline within two ticks (≤ 120 s).
4. **J4 — recertification gate** — Artificially mutate `PromptMemory` beyond the `max-diff-blocks` threshold; running `mvn verify` fails with a structured error naming the drifted block IDs.
5. **J5 — memory-updates SSE channel** — Subscribe to `/api/sessions/sse?channel=memory-updates`; each time a session completes and memory is updated, a `memory-update` SSE event appears within the session's SSE timeline.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-improving-deep-researcher demonstrating the
evaluator-optimizer × research-intel cell. Runs out of the box (no
external services).
Maven group io.akka.samples. Artifact id
evaluator-optimizer-research-intel-akka-self-improving-researcher.
Java package io.akka.samples.selfimprovingdeepresearchagent. Akka 3.6.0.
HTTP port 9745.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ResearchAgent — definition() with
    capability(TaskAcceptance.of(RESEARCH).maxIterationsPerTask(4))
    AND capability(TaskAcceptance.of(REFINE_RESEARCH).maxIterationsPerTask(4)).
    System prompt loaded from prompts/research-agent.md. Returns
    ResearchReport{executiveSummary, evidence, sources, researchedAt} for
    both RESEARCH and REFINE_RESEARCH. The REFINE_RESEARCH task takes
    (originalQuery, priorReport, QualityNotes) as inputs.
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/evaluator-agent.md. Returns
    QualityEvaluation{verdict, notes, score, evaluatedAt} where verdict
    is the EvaluatorVerdict enum (SUFFICIENT | REFINE) and score is a
    1–5 integer rubric.
  * PromptRewriterAgent — definition() with
    capability(TaskAcceptance.of(REWRITE_MEMORY).maxIterationsPerTask(2)).
    System prompt from prompts/prompt-rewriter-agent.md. Returns
    MemoryDiff{changes, triggerSessionId, triggerVerdict, diffedAt}.

- 1 Workflow ResearchSessionWorkflow with steps:
    startStep -> researchStep -> evaluateStep ->
    [verdict SUFFICIENT? acceptStep : (attemptCount < maxAttempts ?
       refineStep -> evaluateStep : maxAttemptsStep)] ->
    rewriteMemoryStep -> END.
  researchStep calls forAutonomousAgent(ResearchAgent.class, sessionId)
    .runSingleTask(RESEARCH) then forTask(taskId).result(RESEARCH).
  refineStep calls forAutonomousAgent(ResearchAgent.class, sessionId)
    .runSingleTask(REFINE_RESEARCH) with (originalQuery, priorReport,
    QualityNotes) then forTask(taskId).result(REFINE_RESEARCH).
  evaluateStep calls forAutonomousAgent(EvaluatorAgent.class, sessionId)
    .runSingleTask(EVALUATE_REPORT). rewriteMemoryStep calls
  forAutonomousAgent(PromptRewriterAgent.class, sessionId)
    .runSingleTask(REWRITE_MEMORY) with (sessionId, finalReport,
    currentPromptMemory, triggerVerdict). acceptStep emits
    SessionAccepted. maxAttemptsStep emits SessionMaxAttemptsReached
    with bestAttemptNumber (highest score), bestReport, and a structured
    reason. rewriteMemoryStep emits MemoryUpdated with the MemoryDiff.
  Override settings() with stepTimeout(90s) on researchStep,
    refineStep, evaluateStep, rewriteMemoryStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(maxAttemptsStep)).

- 1 EventSourcedEntity SessionEntity holding state
    Session{sessionId, topic, depthHint, maxAttempts, SessionStatus status,
    List<ResearchAttempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<ResearchReport> acceptedReport,
    Optional<MemoryDiff> memoryDiff,
    Instant createdAt, Optional<Instant> finishedAt}.
  SessionStatus enum: RESEARCHING, EVALUATING, ACCEPTED,
    MAX_ATTEMPTS_REACHED.
  Events: SessionCreated, AttemptResearched, AttemptEvaluated,
    SessionAccepted, SessionMaxAttemptsReached, MemoryUpdated,
    QualityEvalRecorded, PromptDriftRecorded.
  Commands: createSession, recordAttempt, recordEvaluation,
    acceptSession, reachMaxAttempts, recordMemoryUpdate,
    recordQualityEval, recordDrift, getSession.
  emptyState() returns Session.initial("", "", 3) with no
    commandContext() reference. Event-applier wraps lifecycle fields
    with Optional.of(...).

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(topic,
    depthHint, sourceTypePreference, requestedBy) emitting
    QuerySubmitted{sessionId, topic, depthHint, sourceTypePreference,
    requestedBy, submittedAt}.

- 1 View SessionsView with row type SessionRow (mirrors Session; the
    attempts list is bounded at maxAttempts so size stays reasonable).
  Table updater consumes SessionEntity events. ONE query getAllSessions
    SELECT * AS sessions FROM sessions_view. No WHERE status filter —
    caller filters client-side because Akka cannot auto-index enum
    columns (Lesson 2).

- 1 Consumer QueryConsumer subscribed to QueryQueue events; on
    QuerySubmitted starts a ResearchSessionWorkflow with the sessionId
    as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 90s, reads next line from
    src/main/resources/sample-events/research-queries.jsonl and calls
    QueryQueue.enqueueQuery.
  * DriftSampler — every 60s, reads the current PromptMemory from
    src/main/resources/governance/prompt-memory.json, hashes it, and
    compares to the last recorded fingerprint. If changed, calls
    SessionEntity.recordDrift(oldFingerprint, newFingerprint,
    changedBlockCount). Idempotent per fingerprint transition.

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /sessions, GET /sessions,
    GET /sessions/{id}, GET /sessions/sse (supports
    ?channel=memory-updates for the H1 monitor stream), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
    The POST /sessions body is {topic, depthHint?, sourceTypePreference?,
    requestedBy?}; missing depthHint defaults to "standard", missing
    sourceTypePreference defaults to "any", missing requestedBy defaults
    to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ResearchTasks.java declaring four Task<R> constants: RESEARCH
    (resultConformsTo ResearchReport), REFINE_RESEARCH (ResearchReport),
    EVALUATE_REPORT (QualityEvaluation), REWRITE_MEMORY (MemoryDiff).
- Domain records ResearchQuery, EvidenceSegment, SourceManifest,
    ResearchReport, QualityNotes, QualityEvaluation, MemoryBlock,
    PromptMemory, MemoryBlockChange, MemoryDiff, ResearchAttempt,
    Session; enums SessionStatus, EvaluatorVerdict.
- src/main/resources/application.conf with
    akka.javasdk.dev-mode.http-port = 9745 and
    akka.javasdk.agent model-provider blocks for anthropic
    (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini
    (gemini-2.5-flash), each api-key read from the canonical env vars
    (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
    ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
    self-improving-researcher.session.max-attempts = 3 and
    self-improving-researcher.drift.max-diff-blocks = 2, overridable
    by env var.
- src/main/resources/sample-events/research-queries.jsonl with 8
    canned query lines, each shaped
    {"topic":"...", "depthHint":"standard", "sourceTypePreference":"any"}.
- src/main/resources/governance/prompt-memory.json — initial
    PromptMemory with 4 seed MemoryBlocks (role, strategy, source-prefs,
    synthesis-style). This file is the mutable memory store. The
    baseline fingerprint is copied to
    src/main/resources/governance/prompt-baseline.json at project init.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
    README.md (copies of the root-level files for the endpoint to serve).
- eval-matrix.yaml at the project root with 3 controls (E1
    eval-periodic drift-fairness-watch, S1 ci-gate
    periodic-recertification-gate, H1 hotl deployer-runtime-monitoring)
    and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
    purpose.primary_function = research-synthesis-with-self-improvement,
    decisions.authority_level = advisory-only,
    data.data_classes.pii = false,
    capabilities.content-generation = true,
    capabilities.synthetic-media = false;
    deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/research-agent.md, prompts/evaluator-agent.md,
    prompts/prompt-rewriter-agent.md loaded at agent startup.
- README.md at the project root (this file).
- src/main/resources/static-resources/index.html — a single
    self-contained HTML file (no ui/ folder, no npm build). Inline CSS +
    JS. Runtime CDN imports for markdown and YAML libs are acceptable.
    Five tabs matching the formal exemplar: Overview (eyebrow + headline
    + no subtitle + Try it / How it works / Components / API contract
    cards); Architecture (4 mermaid diagrams + click-to-expand component
    table); Risk Survey (7 sub-tabs from governance.html with answers
    populated from risk-survey.yaml; unanswered .qb opacity 0.45);
    Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source
    table with click-to-expand rows); App UI (form + live list with
    status pills, click-to-expand per-attempt timeline showing each
    report summary, evaluator verdict, quality notes, and memory diff).
    Browser title exactly:
    <title>Akka Sample: Self-Improving Deep Research Agent</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment
    for ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY.
    If exactly one is set, default application.conf's model-provider to
    match and proceed silently.
- If none is set, ask the user how to source the key, offering five
    options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that
        returns random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf.
    (c) Point to an existing env file — record the PATH in a
        project-local .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://,
        vault://; recorded in .akka-build.yaml.
    (e) Type once in this session — value lives in Claude session
        memory; gone when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java
    implementing ModelProvider with a per-agent dispatch on agent class
    name or Task<R> id. Each agent's branch reads a JSON file from
    src/main/resources/mock-responses/<agent-name>.json, picks one
    entry pseudo-randomly, and deserializes it into the agent's typed
    return shape.
- Per-agent mock-response shapes:
    research-agent.json — 6 ResearchReport entries. Four are
      first-pass reports on topics from research-queries.jsonl with
      3–6 EvidenceSegments each. Two are refined reports with more
      evidence segments and a stronger executiveSummary that references
      prior QualityNotes.
    evaluator-agent.json — 6 QualityEvaluation entries. Three return
      verdict=SUFFICIENT with score=4 or 5 and a one-sentence rationale.
      Three return verdict=REFINE with score=2 or 3 and QualityNotes
      of 3–4 bullets ("coverage omits the regulatory angle",
      "evidence segments lack primary source refs",
      "synthesis coherence breaks at paragraph 3",
      "source diversity: all refs from same domain").
    prompt-rewriter-agent.json — 4 MemoryDiff entries. Two represent
      SUFFICIENT sessions (small diffs: 1–2 blocks updated with
      strategy refinements). Two represent REFINE/MAX sessions (larger
      diffs: 3 blocks updated, adding failure-pattern guidance to the
      strategy block).
- A MockModelProvider.seedFor(sessionId, attemptNumber) helper makes
    selection deterministic per (sessionId, attemptNumber) across
    restarts.

Constraints — see
    explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
    for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
    ResearchAgent, EvaluatorAgent, and PromptRewriterAgent all extend
    akka.javasdk.agent.autonomous.AutonomousAgent and ship with a
    ResearchTasks companion declaring the four Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
    stepTimeout(90s) override; the default 5-second timeout is never
    inherited.
- Lesson 6: every nullable lifecycle field on the Session row record
    is Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ResearchTasks.java is mandatory; generating any agent
    without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o,
    gemini-2.5-flash — verified current; do not substitute deprecated
    identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash
    command), never mvn akka:run.
- Lesson 10: HTTP port is 9745, declared in application.conf
    dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI
    never surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no
    horizontal scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" —
    never T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
    overrides AND theme variables for state-diagram label colour,
    edge-label foreignObject overflow:visible,
    transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf
    records only ${?VAR_NAME} substitution; Bootstrap.java fails fast
    if the reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
    NEVER by NodeList index. The DOM contains exactly five
    <section class="tab-panel"> elements; removed panels are deleted
    from the HTML, not hidden with display:none.
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
