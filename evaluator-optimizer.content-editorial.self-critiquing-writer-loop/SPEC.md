# SPEC — self-critiquing-writer-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Critiquing Writer Loop.
**One-line pitch:** Submit a brief; a writer agent drafts a content passage; a critic agent scores it against a quality rubric; the two iterate until the critic accepts the draft or the loop exhausts its iteration budget.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`WriterAgent`) and a reviewer agent (`CriticAgent`), feeding each critique back into the next draft until convergence or a halt. The blueprint demonstrates two governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, and a **budget-cap halt** that ends the loop gracefully at the configured iteration ceiling, firing an observable halt event and preserving every attempt for audit.

## 3. User-facing flows

The user opens the App UI tab and submits a brief (a topic plus an optional quality threshold).

1. The system creates an `Article` record in `DRAFTING` and starts a `RefinementWorkflow`.
2. The Writer drafts attempt #1: a focused prose passage on the brief.
3. The Critic scores the draft against a fixed rubric (clarity, coherence, brief adherence, register appropriateness) and returns either `ACCEPT` with a one-line rationale, or `REVISE` with a typed `CritiqueNotes` payload (three bullets at most).
4. On `ACCEPT`, the workflow transitions the article to `ACCEPTED` with the winning attempt's text and the critic's rationale.
5. On `REVISE`, the workflow records the attempt and critique on the entity, then calls the Writer again with the critique attached. The Writer produces attempt #2.
6. If the loop reaches `maxIterations` (default 4) without an `ACCEPT`, the budget-cap halt activates: the workflow ends with `BUDGET_EXHAUSTED`, the best-scoring attempt is preserved on the entity along with every critique for audit, and a `BudgetExhausted` event fires before the terminal `EvalRecorded` event.

A `RequestSimulator` (TimedAction) drips a canned brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WriterAgent` | `AutonomousAgent` | Drafts a content passage on a brief; accepts prior critique on revision calls. | `RefinementWorkflow` | returns `DraftPassage` to workflow |
| `CriticAgent` | `AutonomousAgent` | Scores a draft against the rubric; returns `ACCEPT` or `REVISE` with notes. | `RefinementWorkflow` | returns `Critique` to workflow |
| `RefinementWorkflow` | `Workflow` | Runs the draft → critique → revise loop; halts at the iteration budget. | `WritingEndpoint`, `DraftRequestConsumer` | `ArticleEntity` |
| `ArticleEntity` | `EventSourcedEntity` | Holds the article lifecycle, every attempt, every critique, and the final outcome. | `RefinementWorkflow` | `ArticlesView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted brief for replay and audit. | `WritingEndpoint`, `RequestSimulator` | `DraftRequestConsumer` |
| `ArticlesView` | `View` | List-of-articles read model. | `ArticleEntity` events | `WritingEndpoint` |
| `DraftRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `RefinementWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample brief every 60 s from `sample-events/writing-briefs.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `ArticlesView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `ArticleEntity` |
| `WritingEndpoint` | `HttpEndpoint` | `/api/articles/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `ArticlesView`, `RequestQueue`, `ArticleEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Brief(String topic, int qualityThreshold, String requestedBy) {}

record DraftPassage(String text, int wordCount, Instant draftedAt) {}

record CritiqueNotes(List<String> bullets, String overallRationale) {}

record Critique(CriticVerdict verdict, CritiqueNotes notes, int score, Instant evaluatedAt) {}

record Attempt(
    int attemptNumber,
    DraftPassage draft,
    Optional<Critique> critique
) {}

record Article(
    String articleId,
    String topic,
    int qualityThreshold,
    int maxIterations,
    ArticleStatus status,
    List<Attempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ArticleStatus { DRAFTING, EVALUATING, ACCEPTED, BUDGET_EXHAUSTED }

enum CriticVerdict { ACCEPT, REVISE }
```

### Events (on `ArticleEntity`)

`ArticleCreated`, `AttemptDrafted`, `AttemptCritiqued`, `ArticleAccepted`, `BudgetExhausted`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/articles` — body `{ topic, qualityThreshold?, requestedBy? }` → `{ articleId }`. Starts a workflow.
- `GET /api/articles` — list all articles. Optional `?status=DRAFTING|EVALUATING|ACCEPTED|BUDGET_EXHAUSTED`.
- `GET /api/articles/{id}` — one article (including every attempt and every critique).
- `GET /api/articles/sse` — server-sent events stream of every article change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Self-Critiquing Writer Loop"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, halt = red).
- **App UI** — form to submit a brief, live list of articles with status pills, click-to-expand per-attempt timeline showing each draft, the critic's verdict, and the critic's notes.

Browser title: `<title>Akka Sample: Self-Critiquing Writer Loop</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every cycle's critique is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, iterationsUsed }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/articles/{id}`.
- **HT1 — budget-cap halt** (`budget-cap`): when the loop reaches `maxIterations` without an `ACCEPT`, the workflow emits `BudgetExhausted` carrying the iteration count, then ends with the best-scoring attempt's text preserved as the best-of and a structured rejection reason. The halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `WriterAgent` → `prompts/writer.md`. Drafts a prose passage on the brief; on a revision call, takes the prior `CritiqueNotes` as input and produces a new draft.
- `CriticAgent` → `prompts/critic.md`. Scores a draft against the fixed rubric; returns `ACCEPT` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a brief; article progresses `DRAFTING` → `EVALUATING` → `ACCEPTED` within the iteration budget; the App UI shows every attempt's draft and critique.
2. **J2 — budget exhaustion** — Submit a brief whose rubric is impossible (test mode forces the Critic to `REVISE` every attempt); article progresses through every iteration and lands in `BUDGET_EXHAUSTED` with the best draft preserved and a structured rejection reason; a `BudgetExhausted` event is visible in the timeline.
3. **J3 — eval-event timeline** — The expanded view of any article shows one `EvalRecorded` event per attempt and one terminal event on completion.
4. **J4 — revision incorporates notes** — After a `REVISE` verdict, the next attempt's draft visibly addresses the critique bullets (confirmed via mock or live LLM output).

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-critiquing-writer-loop demonstrating the
evaluator-optimizer × content-editorial cell. Runs out of the box
(no external services).
Maven group io.akka.samples. Artifact id
evaluator-optimizer-content-editorial-self-critiquing-writer-loop.
Java package io.akka.samples.selfcritiquingwriterloop.
Akka 3.6.0. HTTP port 9235.

Components to wire (exactly):
- 2 AutonomousAgents:
  * WriterAgent — definition() with
    capability(TaskAcceptance.of(DRAFT).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_DRAFT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/writer.md. Returns DraftPassage{text,
    wordCount, draftedAt} for both DRAFT and REVISE_DRAFT. The
    REVISE_DRAFT task takes (originalBrief, priorDraft, CritiqueNotes) as
    inputs.
  * CriticAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)). System
    prompt from prompts/critic.md. Returns Critique{verdict, notes, score,
    evaluatedAt} where verdict is the CriticVerdict enum (ACCEPT | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow RefinementWorkflow with steps:
    startStep -> draftStep -> critiqueStep ->
    [verdict ACCEPT? acceptStep :
      (iterationCount < maxIterations ?
         draftStep with critique attached : budgetExhaustedStep)] -> END.
  draftStep calls forAutonomousAgent(WriterAgent.class, articleId)
    .runSingleTask(DRAFT or REVISE_DRAFT) then forTask(taskId)
    .result(DRAFT or REVISE_DRAFT). critiqueStep calls
    forAutonomousAgent(CriticAgent.class, articleId).runSingleTask(EVALUATE).
  acceptStep emits ArticleAccepted.
  budgetExhaustedStep emits BudgetExhausted{iterationsUsed, bestAttemptNumber,
    bestText} followed by EvalRecorded carrying the loop-level outcome, then
    ends with status BUDGET_EXHAUSTED. Override settings() with
    stepTimeout(60s) on draftStep and critiqueStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(budgetExhaustedStep)).

- 1 EventSourcedEntity ArticleEntity holding state Article{articleId, topic,
  qualityThreshold, maxIterations, ArticleStatus status, List<Attempt> attempts,
  Optional<Integer> acceptedAttemptNumber, Optional<String> acceptedText,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. ArticleStatus enum: DRAFTING, EVALUATING, ACCEPTED,
  BUDGET_EXHAUSTED. Events: ArticleCreated, AttemptDrafted, AttemptCritiqued,
  ArticleAccepted, BudgetExhausted, EvalRecorded. Commands: createArticle,
  recordDraft, recordCritique, accept, budgetExhausted, recordEval, getArticle.
  emptyState() returns Article.initial("", "", 3, 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueBrief(topic,
  qualityThreshold, requestedBy) emitting BriefSubmitted{articleId, topic,
  qualityThreshold, requestedBy, submittedAt}.

- 1 View ArticlesView with row type ArticleRow (mirrors Article; the attempts
  list is preserved as-is — the list is bounded at maxIterations so size
  stays reasonable). Table updater consumes ArticleEntity events. ONE query
  getAllArticles SELECT * AS articles FROM articles_view. No WHERE status
  filter — caller filters client-side because Akka cannot auto-index enum
  columns (Lesson 2).

- 1 Consumer DraftRequestConsumer subscribed to RequestQueue events; on
  BriefSubmitted starts a RefinementWorkflow with the articleId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/writing-briefs.jsonl and calls
    RequestQueue.enqueueBrief.
  * EvalSampler — every 30s, queries ArticlesView.getAllArticles, finds articles
    with a critiqued attempt that has not yet been recorded as an
    EvalRecorded event, and calls ArticleEntity.recordEval(attemptNumber,
    verdict, score, iterationsUsed). Idempotent per (articleId, attemptNumber).

- 2 HttpEndpoints:
  * WritingEndpoint at /api with POST /articles, GET /articles,
    GET /articles/{id}, GET /articles/sse, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/. The
    POST /articles body is {topic, qualityThreshold?, requestedBy?}; missing
    qualityThreshold defaults to 3 (on a 1–5 rubric), missing requestedBy
    defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- WritingTasks.java declaring three Task<R> constants: DRAFT (resultConformsTo
  DraftPassage), REVISE_DRAFT (DraftPassage), EVALUATE (Critique).
- Domain records DraftPassage, CritiqueNotes, Critique, Attempt, Article;
  enums ArticleStatus, CriticVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9235 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from the canonical
  env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  self-critiquing-writer-loop.refinement.max-iterations = 4 and
  self-critiquing-writer-loop.refinement.default-quality-threshold = 3,
  overridable by env var.
- src/main/resources/sample-events/writing-briefs.jsonl with 8 canned brief
  lines, each shaped {"topic":"...", "qualityThreshold":3}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-event
  on-decision-eval, HT1 halt budget-cap) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = content-writing-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/writer.md, prompts/critic.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Self-Critiquing Writer Loop",
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
  <title>Akka Sample: Self-Critiquing Writer Loop</title>.

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
  named in Section 9: writer.json, critic.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    writer.json — 6 DraftPassage entries. Three are "first-pass" passages
      between 80 and 140 words on topics in writing-briefs.jsonl (clear
      prose, neutral register). Two are "revision" passages that reference
      the prior critique (tightened focus, removed the flagged section).
      One is a deliberately unfocused passage that exercises the
      below-threshold path in J2.
    critic.json — 6 Critique entries. Three return verdict=ACCEPT with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a CritiqueNotes payload of
      three bullets ("opening paragraph lacks a clear claim",
      "register shifts between formal and casual in paragraph 2",
      "closing does not return to the brief's central question").
- A MockModelProvider.seedFor(articleId, attemptNumber) helper makes the
  selection deterministic per (articleId, attemptNumber) so the same
  article in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. WriterAgent
  and CriticAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a WritingTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Article row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: WritingTasks.java is mandatory; generating WriterAgent or
  CriticAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9235, declared in application.conf
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
