# SPEC — reflexion-self-critique

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Reflexion Self-Critique.
**One-line pitch:** Submit a research question; an actor agent retrieves tool-grounded citations and drafts an answer; a reflexion agent scores it against coverage, accuracy, and citation completeness; verbal reinforcement memory conditions the next attempt until the answer passes or the retry ceiling is reached.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern where each iteration produces an explicit verbal memory trace — not just structured feedback, but a natural-language reinforcement note that the actor reads as prior context on every retry. The blueprint wires three governance mechanisms: an **eval-event** that records every cycle's reflexion verdict for downstream quality measurement, an **output guardrail** that gates each answer against a deterministic citation-floor check before the reflexion agent runs, and a **halt** that ends the loop at the retry ceiling without leaving the query in a degenerate state.

The citation retrieval is modeled as a stub tool — the actor calls a `searchDocuments(query, topK)` function that returns `SourceDocument` records from an in-memory corpus. Real deployments swap the stub for a vector-store or web-search client without changing the agent or workflow.

## 3. User-facing flows

The user opens the App UI tab and submits a research question (optional: a citation floor).

1. The system creates a `Query` record in `RESEARCHING` and starts a `ReflexionWorkflow`.
2. The Actor searches for source documents (tool call), reads the top results, and drafts an answer grounded in the retrieved citations.
3. The output guardrail checks that the answer cites at least the citation floor (default 2). Under-cited answers are short-circuited back to the Actor with a deterministic feedback note; they never reach the Reflexion agent.
4. The Reflexion agent scores the answer against a four-dimension rubric (citation count, coverage, accuracy self-check, source quality) and returns either `PASS` with a one-sentence rationale, or `RETRY` with a typed `ReflexionNote` payload (a verbal reinforcement paragraph plus three focus bullets).
5. On `PASS`, the workflow transitions the query to `RESOLVED` with the winning answer and the reflexion agent's rationale.
6. On `RETRY`, the workflow records the attempt, the guardrail verdict, the reflexion note, and the verdict on the entity, then calls the Actor again with the full `ReflexionNote` as verbal memory. The Actor produces attempt #2, now conditioned on the prior note.
7. If the loop reaches `maxAttempts` (default 4) without a `PASS`, the halt mechanism activates: the workflow ends with `EXHAUSTED`, the best-scoring attempt is preserved on the entity along with every note for audit, and a `ReflexionRecorded` event captures the terminal outcome.

A `QuerySimulator` (TimedAction) drips a canned question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ActorAgent` | `AutonomousAgent` | Retrieves source documents via tool call; synthesizes a citation-grounded answer; on retry, accepts prior `ReflexionNote` as verbal memory. | `ReflexionWorkflow` | returns `CandidateAnswer` to workflow |
| `ReflexionAgent` | `AutonomousAgent` | Scores an answer against the rubric; returns `PASS` or `RETRY` with a verbal reinforcement note. | `ReflexionWorkflow` | returns `Reflexion` to workflow |
| `ReflexionWorkflow` | `Workflow` | Runs the answer → guardrail → critique → revise loop; halts at the ceiling. | `ResearchEndpoint`, `QueryConsumer` | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Holds the query lifecycle, every attempt, every reflexion note, and the final outcome. | `ReflexionWorkflow` | `QueriesView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted question for replay and audit. | `ResearchEndpoint`, `QuerySimulator` | `QueryConsumer` |
| `QueriesView` | `View` | List-of-queries read model. | `QueryEntity` events | `ResearchEndpoint` |
| `QueryConsumer` | `Consumer` | Subscribes to `QueryQueue` events; starts a workflow per submission. | `QueryQueue` events | `ReflexionWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample question every 60 s from `sample-events/research-questions.jsonl`. | scheduler | `QueryQueue` |
| `ReflexionSampler` | `TimedAction` | Every 30 s, scans `QueriesView`, records a `ReflexionRecorded` event for any cycle that completed since the last tick. | scheduler | `QueryEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `QueriesView`, `QueryQueue`, `QueryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ResearchQuestion(String text, int citationFloor, String submittedBy) {}

record SourceDocument(String docId, String title, String snippet, String url) {}

record CandidateAnswer(
    String text,
    List<SourceDocument> citations,
    int citationCount,
    Instant answeredAt
) {}

record CitationGuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ReflexionNote(
    String reinforcementParagraph,
    List<String> focusBullets,
    String overallRationale
) {}

record Reflexion(
    ReflexionVerdict verdict,
    ReflexionNote note,
    int score,
    Instant reflectedAt
) {}

record Attempt(
    int attemptNumber,
    CandidateAnswer answer,
    CitationGuardrailVerdict guardrail,
    Optional<Reflexion> reflexion
) {}

record Query(
    String queryId,
    String questionText,
    int citationFloor,
    int maxAttempts,
    QueryStatus status,
    List<Attempt> attempts,
    Optional<Integer> resolvedAttemptNumber,
    Optional<String> resolvedAnswerText,
    Optional<String> exhaustionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus { RESEARCHING, REFLECTING, RESOLVED, EXHAUSTED }

enum ReflexionVerdict { PASS, RETRY }
```

### Events (on `QueryEntity`)

`QueryCreated`, `AttemptAnswered`, `AttemptCitationGuardrailRecorded`, `AttemptReflected`, `QueryResolved`, `QueryExhausted`, `ReflexionRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/queries` — body `{ questionText, citationFloor?, submittedBy? }` → `{ queryId }`. Starts a workflow.
- `GET /api/queries` — list all queries. Optional `?status=RESEARCHING|REFLECTING|RESOLVED|EXHAUSTED`.
- `GET /api/queries/{id}` — one query (including every attempt and every reflexion note).
- `GET /api/queries/sse` — server-sent events stream of every query change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Reflexion Self-Critique"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, halt = red).
- **App UI** — form to submit a question, live list of queries with status pills, click-to-expand per-attempt timeline showing each answer, the citation guardrail verdict, the reflexion verdict, and the verbal reinforcement note.

Browser title: `<title>Akka Sample: Reflexion Self-Critique</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ActorAgent`): a deterministic check that the answer's `citationCount >= citationFloor`. Under-cited answers short-circuit back to the Actor with a structured feedback note (`reasonCode = UNDER_CITED`); they never reach the Reflexion agent. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's reflexion is recorded as a `ReflexionRecorded` event with `{ attemptNumber, verdict, score, citationShortfall }`. The `ReflexionSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/queries/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without a `PASS`, the workflow ends with `QueryExhausted`. The entity preserves every attempt, every reflexion note, the best-scoring attempt's answer, and a structured exhaustion reason. The system never discards attempts or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `ActorAgent` → `prompts/actor.md`. Retrieves source documents via `searchDocuments` tool call; synthesizes a cited answer; on a retry call, takes the prior `ReflexionNote` as verbal memory and produces a revised answer that addresses the reinforcement paragraph and focus bullets.
- `ReflexionAgent` → `prompts/reflexion.md`. Scores an answer against the fixed rubric; returns `PASS` with a one-sentence rationale or `RETRY` with a verbal reinforcement paragraph plus three focus bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a question; query progresses `RESEARCHING` → `REFLECTING` → `RESOLVED` within the retry ceiling; the App UI shows every attempt's answer, citation list, and reflexion note.
2. **J2 — halt at ceiling** — Submit a question whose rubric is impossible (test mode forces `RETRY` every attempt); query progresses through every attempt and lands in `EXHAUSTED` with the best answer preserved and a structured exhaustion reason.
3. **J3 — guardrail block** — Submit a question with `citationFloor = 3`; the Actor's first answer cites fewer than 3 sources; the guardrail short-circuits with `reasonCode = UNDER_CITED`, the Actor re-answers with more citations, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any query shows one `ReflexionRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named reflexion-self-critique demonstrating the evaluator-optimizer ×
research-intel cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-research-intel-reflexion-self-critique.
Java package io.akka.samples.reflexion. Akka 3.6.0. HTTP port 9547.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ActorAgent — definition() with
    capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(4))
    AND capability(TaskAcceptance.of(RETRY_ANSWER).maxIterationsPerTask(4)).
    System prompt loaded from prompts/actor.md. Returns CandidateAnswer{text,
    citations, citationCount, answeredAt} for both ANSWER and RETRY_ANSWER.
    The RETRY_ANSWER task takes (originalQuestion, priorAnswer, ReflexionNote)
    as inputs. The agent calls a stub tool searchDocuments(query, topK) that
    reads from an in-memory corpus of SourceDocument records loaded from
    src/main/resources/sample-events/source-corpus.jsonl.
  * ReflexionAgent — definition() with
    capability(TaskAcceptance.of(REFLECT).maxIterationsPerTask(2)). System
    prompt from prompts/reflexion.md. Returns Reflexion{verdict, note, score,
    reflectedAt} where verdict is the ReflexionVerdict enum (PASS | RETRY)
    and score is a 1–5 integer rubric.

- 1 Workflow ReflexionWorkflow with steps:
    startStep -> answerStep -> guardrailStep -> [guardrail FAIL? answerStep
    again with structured feedback : reflectStep] ->
    [verdict PASS? resolveStep : (attemptCount < maxAttempts ?
       answerStep with note as verbal memory : exhaustStep)] -> END.
  answerStep calls forAutonomousAgent(ActorAgent.class, queryId).runSingleTask(
    ANSWER or RETRY_ANSWER) then forTask(taskId).result(ANSWER or RETRY_ANSWER).
  reflectStep calls forAutonomousAgent(ReflexionAgent.class, queryId).runSingleTask(REFLECT).
  resolveStep emits QueryResolved with the accepted answer and reflexion rationale.
  exhaustStep emits QueryExhausted with the highest-scoring attempt's text as
    best-of and a structured exhaustionReason ("max attempts reached (N)").
  Override settings() with stepTimeout(60s) on answerStep and reflectStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    answer.citationCount() >= citationFloor. On FAIL, emits
    AttemptCitationGuardrailRecorded with verdict.passed = false and
    reasonCode = "UNDER_CITED" and detail naming the shortfall, then
    transitions back to answerStep with a structured feedback ReflexionNote
    containing reinforcementParagraph("Answer does not meet the citation floor;
    retrieve more sources and resubmit.") and a single focusBullet("Minimum
    citations required: N; found: M.").

- 1 EventSourcedEntity QueryEntity holding state Query{queryId, questionText,
  citationFloor, maxAttempts, QueryStatus status, List<Attempt> attempts,
  Optional<Integer> resolvedAttemptNumber, Optional<String> resolvedAnswerText,
  Optional<String> exhaustionReason, Instant createdAt, Optional<Instant>
  finishedAt}. QueryStatus enum: RESEARCHING, REFLECTING, RESOLVED, EXHAUSTED.
  Events: QueryCreated, AttemptAnswered, AttemptCitationGuardrailRecorded,
  AttemptReflected, QueryResolved, QueryExhausted, ReflexionRecorded.
  Commands: createQuery, recordAnswer, recordCitationGuardrail, recordReflexion,
  resolve, exhaust, recordReflexionEval, getQuery. emptyState() returns
  Query.initial("", "", 2, 4) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity QueryQueue with command enqueueQuestion(questionText,
  citationFloor, submittedBy) emitting QuestionSubmitted{queryId, questionText,
  citationFloor, submittedBy, submittedAt}.

- 1 View QueriesView with row type QueryRow (mirrors Query; the attempts list
  is preserved as-is — bounded at maxAttempts so size stays reasonable).
  Table updater consumes QueryEntity events. ONE query getAllQueries
  SELECT * AS queries FROM queries_view. No WHERE status filter — caller filters
  client-side because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer QueryConsumer subscribed to QueryQueue events; on QuestionSubmitted
  starts a ReflexionWorkflow with the queryId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-questions.jsonl and calls
    QueryQueue.enqueueQuestion.
  * ReflexionSampler — every 30s, queries QueriesView.getAllQueries, finds queries
    with a reflected attempt that has not yet been recorded as a
    ReflexionRecorded event, and calls QueryEntity.recordReflexionEval(
    attemptNumber, verdict, score, citationShortfall). Idempotent per
    (queryId, attemptNumber).

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /queries, GET /queries, GET /queries/{id},
    GET /queries/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /queries body
    is {questionText, citationFloor?, submittedBy?}; missing citationFloor
    defaults to 2, missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ResearchTasks.java declaring three Task<R> constants: ANSWER (resultConformsTo
  CandidateAnswer), RETRY_ANSWER (CandidateAnswer), REFLECT (Reflexion).
- Domain records CandidateAnswer, SourceDocument, CitationGuardrailVerdict,
  ReflexionNote, Reflexion, Attempt, Query; enums QueryStatus, ReflexionVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9547 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  reflexion.max-attempts = 4 and
  reflexion.default-citation-floor = 2, overridable by env var.
- src/main/resources/sample-events/research-questions.jsonl with 8 canned
  question lines, each shaped {"questionText":"...", "citationFloor":2}.
- src/main/resources/sample-events/source-corpus.jsonl — 30 SourceDocument
  records spanning 5 research topics (climate science, healthcare AI, supply
  chain logistics, financial regulation, cybersecurity policy); the stub
  searchDocuments tool scores by simple keyword overlap.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) plus HT1 halt
  graceful-degradation, and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = research-synthesis-with-citations,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/actor.md, prompts/reflexion.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Reflexion Self-Critique",
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
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Reflexion Self-Critique</title>.

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
  named in Section 9: actor.json, reflexion.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    actor.json — 6 CandidateAnswer entries. Three are well-cited answers
      with 3–5 SourceDocument citations on topics from research-questions.jsonl.
      Two are "retry" answers that reference the prior reflexion note (revised
      framing, additional sources). One is an intentionally under-cited answer
      (1 citation) used to exercise the guardrail in J3.
    reflexion.json — 6 Reflexion entries. Three return verdict=PASS with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=RETRY with score=2 or 3 and a ReflexionNote payload with a
      reinforcementParagraph and three focusBullets ("coverage of the 2023
      policy update is absent", "source quality is weak — no peer-reviewed
      citations", "accuracy of the second claim is unverified").
- A MockModelProvider.seedFor(queryId, attemptNumber) helper makes the
  selection deterministic per (queryId, attemptNumber) so the same query in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. ActorAgent
  and ReflexionAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a ResearchTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Query row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ResearchTasks.java is mandatory; generating ActorAgent or
  ReflexionAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9547, declared in application.conf
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
