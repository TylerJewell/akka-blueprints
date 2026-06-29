# SPEC — memory-eval-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Evals with Memory.
**One-line pitch:** Ask a question; an answer agent retrieves relevant memory entries and produces a cited response; a scorer agent grades the answer against a quality rubric; the two iterate until the scorer accepts or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`AnswerAgent`) and a reviewer agent (`ScorerAgent`), feeding each quality verdict back into the next answer attempt until convergence or a halt. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every cycle's quality verdict for downstream measurement, a **periodic eval** (drift-fairness-watch) that monitors whether answer quality degrades as memory accumulates across sessions, and a **PII sanitizer** that scrubs user-contributed content before it is written to the long-term memory store.

## 3. User-facing flows

The user opens the App UI tab and submits a question (free text).

1. The system creates a `Session` record in `ANSWERING` and starts an `AnswerWorkflow`.
2. The `AnswerAgent` retrieves the top-K memory entries for the user from `MemoryView` and produces attempt #1: a cited answer grounded in those entries.
3. The `ScorerAgent` scores the answer against a fixed rubric (relevance, grounding, completeness, coherence) and returns either `PASS` with a one-line rationale, or `IMPROVE` with a typed `ScoringNotes` payload (three bullets at most).
4. On `PASS`, the workflow transitions the session turn to `ACCEPTED` with the winning answer and the scorer's rationale.
5. On `IMPROVE`, the workflow records the attempt, the score, and the feedback on the entity, then calls the `AnswerAgent` again with the notes attached. The agent produces attempt #2 incorporating the feedback.
6. If the loop reaches `maxAttempts` (default 3) without a `PASS`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring attempt is preserved on the entity along with every score for audit, and an `EvalRecorded` event captures the rejection.
7. After the turn ends (either terminal state), the workflow writes any new information from the accepted answer back to `MemoryStore` (post-sanitization).

A `QuestionSimulator` (TimedAction) drips a canned question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AnswerAgent` | `AutonomousAgent` | Retrieves relevant memory entries and produces a cited answer; on a retry call, incorporates prior scorer feedback. | `AnswerWorkflow` | returns `AnswerAttempt` to workflow |
| `ScorerAgent` | `AutonomousAgent` | Scores an answer against the rubric; returns `PASS` or `IMPROVE` with notes. | `AnswerWorkflow` | returns `Score` to workflow |
| `AnswerWorkflow` | `Workflow` | Runs the answer → score → revise loop; halts at the ceiling; writes memory entries post-turn. | `SessionEndpoint`, `QuestionConsumer` | `SessionEntity`, `MemoryStore` |
| `SessionEntity` | `EventSourcedEntity` | Holds the session turn lifecycle, every answer attempt, every score, and the final outcome. | `AnswerWorkflow` | `SessionView` |
| `MemoryStore` | `EventSourcedEntity` | Accumulates sanitized memory entries for a user; PII is stripped before write. | `AnswerWorkflow` (post-turn write), `SessionEndpoint` | `MemoryView` |
| `SessionView` | `View` | List-of-session-turns read model. | `SessionEntity` events | `SessionEndpoint` |
| `MemoryView` | `View` | Per-user memory entry read model; queried by `AnswerAgent` retrieval step. | `MemoryStore` events | `AnswerWorkflow` |
| `QuestionConsumer` | `Consumer` | Subscribes to `SessionEntity` question-submitted events; starts a workflow per question. | `SessionEntity` events | `AnswerWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample question every 60 s from `sample-events/questions.jsonl`. | scheduler | `SessionEndpoint` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `SessionView`, records an `EvalRecorded` event for any scored attempt not yet sampled. | scheduler | `SessionEntity` |
| `DriftWatcher` | `TimedAction` | Every 10 min, computes a rolling drift metric across recent session turns and emits a `DriftCheckRecorded` event. | scheduler | `SessionEntity` |
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit question, get, list, SSE; plus `/api/memory/*` and `/api/metadata/*`. | — | `SessionView`, `MemoryView`, `SessionEntity`, `MemoryStore` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Question(String questionId, String text, String userId) {}

record MemoryEntry(String entryId, String userId, String content, String sourceSessionId, Instant recordedAt) {}

record ScoringNotes(List<String> bullets, String overallRationale) {}

record Score(ScorerVerdict verdict, ScoringNotes notes, int score, Instant scoredAt) {}

record AnswerAttempt(
    int attemptNumber,
    String answerText,
    List<String> citedEntryIds,
    Optional<Score> score,
    Instant answeredAt
) {}

record Session(
    String sessionId,
    String userId,
    String questionText,
    int maxAttempts,
    SessionStatus status,
    List<AnswerAttempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedAnswer,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { ANSWERING, SCORING, ACCEPTED, REJECTED_FINAL }

enum ScorerVerdict { PASS, IMPROVE }
```

### Events (on `SessionEntity`)

`SessionCreated`, `AnswerAttempted`, `AttemptScored`, `SessionAccepted`, `SessionRejectedFinal`, `EvalRecorded`, `DriftCheckRecorded`.

See `reference/data-model.md` for the full table.

### Events (on `MemoryStore`)

`MemoryEntryAdded`, `MemoryEntryPiiRedacted`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ userId?, text }` → `{ sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all session turns. Optional `?status=ANSWERING|SCORING|ACCEPTED|REJECTED_FINAL`.
- `GET /api/sessions/{id}` — one session turn (including every attempt and every score).
- `GET /api/sessions/sse` — server-sent events stream of every session turn change.
- `GET /api/memory/{userId}` — list memory entries for a user.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Evals with Memory"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, eval-periodic = blue, sanitizer = red).
- **App UI** — form to submit a question, live list of session turns with status pills, click-to-expand per-attempt timeline showing each answer, the scorer's verdict, and the scorer's notes; plus a memory panel showing entries for the current user.

Browser title: `<title>Akka Sample: Evals with Memory</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every cycle's scorer verdict is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, userId }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking.
- **E2 — eval-periodic** (`drift-fairness-watch`): every 10 minutes, `DriftWatcher` computes a rolling pass-rate and average-score across recent session turns and emits a `DriftCheckRecorded` event. If the rolling pass-rate drops below a configurable threshold (`evals-with-memory.drift.pass-rate-floor`, default 0.60), the event carries `driftFlagged = true`. Enforcement: non-blocking.
- **S1 — sanitizer** (`pii`): before any content is written to `MemoryStore`, the workflow applies a deterministic PII-detection pass (email, phone, SSN patterns). Detected values are replaced with a redaction token (`[REDACTED:<type>]`); the original value is never persisted. Enforcement: system-level.

## 9. Agent prompts

- `AnswerAgent` → `prompts/answer-agent.md`. Retrieves memory entries and produces a cited answer; on a retry call, takes the prior `ScoringNotes` as input and revises the answer.
- `ScorerAgent` → `prompts/scorer-agent.md`. Scores an answer against the fixed rubric; returns `PASS` with a one-line rationale or `IMPROVE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a question; session turn progresses `ANSWERING` → `SCORING` → `ACCEPTED` within the retry ceiling; the App UI shows every attempt's answer and score.
2. **J2 — halt at ceiling** — Submit a question whose rubric is impossible to satisfy (test mode forces the Scorer to `IMPROVE` every attempt); session turn lands in `REJECTED_FINAL` with the best answer preserved and a structured rejection reason.
3. **J3 — PII sanitization** — Submit a question whose answer attempt would cause the agent to include an email address in the memory write; the sanitizer redacts it before `MemoryEntryAdded` is emitted; `GET /api/memory/{userId}` never returns the raw value.
4. **J4 — drift check** — After at least 5 session turns have completed, the `DriftWatcher` fires and a `DriftCheckRecorded` event appears in the App UI's drift panel with pass-rate and average-score populated.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named memory-eval-loop demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-memory-eval-loop.
Java package io.akka.samples.evalswithmemory. Akka 3.6.0. HTTP port 9774.

Components to wire (exactly):
- 2 AutonomousAgents:
  * AnswerAgent — definition() with
    capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ANSWER).maxIterationsPerTask(3)).
    System prompt loaded from prompts/answer-agent.md. Returns AnswerAttempt{
    attemptNumber, answerText, citedEntryIds, Optional<Score> score, answeredAt}
    (score is empty at draft time; populated by scorer step). The REVISE_ANSWER
    task takes (questionText, priorAnswer, retrievedEntries, ScoringNotes) as inputs.
  * ScorerAgent — definition() with
    capability(TaskAcceptance.of(SCORE_ANSWER).maxIterationsPerTask(2)). System
    prompt from prompts/scorer-agent.md. Returns Score{verdict, notes, score,
    scoredAt} where verdict is the ScorerVerdict enum (PASS | IMPROVE) and
    score is a 1–5 integer rubric.

- 1 Workflow AnswerWorkflow with steps:
    startStep -> retrieveStep -> answerStep -> scoreStep ->
    [verdict PASS? acceptStep : (attemptCount < maxAttempts ?
       answerStep with notes attached : rejectStep)] ->
    memoryWriteStep -> END.
  retrieveStep calls MemoryView.getEntriesForUser(userId, topK=10); result
    is passed to answerStep. answerStep calls
    forAutonomousAgent(AnswerAgent.class, sessionId).runSingleTask(ANSWER or
    REVISE_ANSWER). scoreStep calls
    forAutonomousAgent(ScorerAgent.class, sessionId).runSingleTask(SCORE_ANSWER).
    acceptStep emits SessionAccepted. rejectStep emits SessionRejectedFinal
    with the highest-scoring attempt's answer as best-of and a structured
    rejectionReason. memoryWriteStep applies the PII sanitizer (S1) to the
    accepted answer text and calls MemoryStore.addEntry for each new fact
    fragment. Override settings() with stepTimeout(60s) on answerStep and
    scoreStep, and defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).

- 2 EventSourcedEntities:
  * SessionEntity holding state Session{sessionId, userId, questionText,
    maxAttempts, SessionStatus status, List<AnswerAttempt> attempts,
    Optional<Integer> acceptedAttemptNumber, Optional<String> acceptedAnswer,
    Optional<String> rejectionReason, Instant createdAt,
    Optional<Instant> finishedAt}. SessionStatus enum: ANSWERING, SCORING,
    ACCEPTED, REJECTED_FINAL. Events: SessionCreated, AnswerAttempted,
    AttemptScored, SessionAccepted, SessionRejectedFinal, EvalRecorded,
    DriftCheckRecorded. Commands: createSession, recordAnswer, recordScore,
    accept, rejectFinal, recordEval, recordDriftCheck, getSession.
    emptyState() returns Session.initial("","","anonymous",3) with no
    commandContext() reference. Event-applier wraps lifecycle fields with
    Optional.of(...).
  * MemoryStore holding state per userId: List<MemoryEntry{entryId, userId,
    content, sourceSessionId, recordedAt}>. Events: MemoryEntryAdded,
    MemoryEntryPiiRedacted. Commands: addEntry(userId, content,
    sourceSessionId), getEntries(userId).

- 2 Views:
  * SessionView with row type SessionRow (mirrors Session; attempts list
    bounded at maxAttempts). ONE query getAllSessions SELECT * FROM
    sessions_view. No WHERE clause — caller filters client-side (Lesson 2).
  * MemoryView with row type MemoryRow (mirrors MemoryEntry). ONE query
    getEntriesForUser(userId) SELECT * FROM memory_view WHERE userId = :userId.

- 1 Consumer QuestionConsumer subscribed to SessionEntity events; on
  SessionCreated starts an AnswerWorkflow with the sessionId as the
  workflow id.

- 3 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/questions.jsonl and calls
    SessionEndpoint POST /api/sessions.
  * EvalSampler — every 30s, queries SessionView.getAllSessions, finds
    attempts that have been scored but do not yet have a matching
    EvalRecorded event, and calls SessionEntity.recordEval(attemptNumber,
    verdict, score, userId). Idempotent per (sessionId, attemptNumber).
  * DriftWatcher — every 10 min, queries SessionView.getAllSessions,
    computes rolling pass-rate (last 20 completed turns) and average
    score, then calls SessionEntity.recordDriftCheck(passRate, avgScore,
    driftFlagged, computedAt) on a sentinel entity id
    "drift-watch-singleton". driftFlagged = passRate < evals-with-memory
    .drift.pass-rate-floor (default 0.60).

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions, GET /sessions, GET
    /sessions/{id}, GET /sessions/sse, GET /memory/{userId}, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. POST /sessions body is
    {text, userId?}; missing userId defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- EvalTasks.java declaring three Task<R> constants: ANSWER (resultConformsTo
  AnswerAttempt), REVISE_ANSWER (AnswerAttempt), SCORE_ANSWER (Score).
- Domain records AnswerAttempt, MemoryEntry, Question, ScoringNotes, Score,
  Session; enums SessionStatus, ScorerVerdict.
- PiiSanitizer.java — pure-function utility; detects email, phone (E.164),
  and SSN patterns via regex; replaces each match with [REDACTED:EMAIL],
  [REDACTED:PHONE], [REDACTED:SSN].
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9774 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  evals-with-memory.answer.max-attempts = 3 and
  evals-with-memory.drift.pass-rate-floor = 0.60, overridable by env var.
- src/main/resources/sample-events/questions.jsonl with 8 canned question
  lines, each shaped {"text":"...", "userId":"demo-user"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (E1 eval-event
  on-decision-eval, E2 eval-periodic drift-fairness-watch, S1 sanitizer
  pii) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling
  purpose.primary_function = question-answering-with-memory,
  decisions.authority_level = advisory-only,
  data.data_classes.pii = true (long-term memory may contain PII),
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/answer-agent.md, prompts/scorer-agent.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Evals with Memory",
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
  per-attempt timeline, plus memory panel). Browser title exactly:
  <title>Akka Sample: Evals with Memory</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java per-agent dispatch on agent class name.
  Per-agent mock-response shapes:
    answer-agent.json — 6 AnswerAttempt entries. Three are first-pass
      answers with 2–3 cited entryIds and 2–4 sentence answers grounded
      in the retrieved memory. Two are revision answers that address the
      prior scoring notes. One deliberately omits citations (used to
      exercise the grounding dimension of the rubric).
    scorer-agent.json — 6 Score entries. Three return verdict=PASS with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=IMPROVE with score=2 or 3 and a ScoringNotes payload of
      three bullets.
- MockModelProvider.seedFor(sessionId, attemptNumber) makes the selection
  deterministic per (sessionId, attemptNumber).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AnswerAgent and ScorerAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with an
  EvalTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override.
- Lesson 6: every nullable lifecycle field on the Session row record is
  Optional<T>; event-applier wraps values with Optional.of(...).
- Lesson 7: EvalTasks.java is mandatory.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: the run command is /akka:build.
- Lesson 10: HTTP port is 9774, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal.
- Lesson 12: the App UI fits the 1080px content column.
- Lesson 13: integration tier is shown as "Runs out of the box".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute.
  The DOM contains exactly five <section class="tab-panel"> elements.
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
