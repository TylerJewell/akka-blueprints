# SPEC — seo-rubric-critic

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SEO Article Evaluation.
**One-line pitch:** Submit a blog post; a rubric agent scores it against a 21-point helpful-content checklist; an optimizer agent revises it; the two iterate until the rubric passes or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a scorer agent (`RubricAgent`) and a revision agent (`OptimizerAgent`), feeding each rubric score back into the next draft until the article clears the acceptance threshold or a halt kicks in. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every round's rubric score for downstream quality measurement, an **output guardrail** that gates each revised draft against a deterministic word-count ceiling before the rubric runs, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the article in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits an article (a title, body text, and a target keyword).

1. The system creates an `Article` record in `SCORING` and starts an `AuditWorkflow`.
2. The RubricAgent scores the article's initial draft against the 21-point checklist and returns a `RubricScore` — a per-dimension breakdown plus an overall pass/fail verdict and a composite score from 0–100.
3. If the score meets the acceptance threshold (default 75/100) and all mandatory dimensions pass, the workflow transitions the article to `APPROVED`.
4. Otherwise the workflow enters the revision loop:
   a. The output guardrail vets the draft's word count against the per-article ceiling. Over-length drafts are short-circuited back to the Optimizer with a deterministic feedback note; they never reach the Rubric.
   b. The OptimizerAgent receives the current draft plus the `RubricFeedback` payload (failing dimensions + improvement notes) and produces a revised `ArticleDraft`.
   c. The RubricAgent re-scores the revised draft.
5. On passing the threshold, the workflow transitions to `APPROVED` with the winning draft and the final `RubricScore`.
6. If the loop reaches `maxRounds` (default 4) without passing, the halt activates: the workflow ends with `FAILED_FINAL`, the highest-scoring draft is preserved on the entity along with every round's score for audit.

An `ArticleSimulator` (TimedAction) drips a canned article brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RubricAgent` | `AutonomousAgent` | Scores a blog post against the 21-point checklist; returns `RubricScore` with pass/fail verdict and per-dimension breakdown. | `AuditWorkflow` | returns `RubricScore` to workflow |
| `OptimizerAgent` | `AutonomousAgent` | Revises specific article sections based on `RubricFeedback`; returns `ArticleDraft`. | `AuditWorkflow` | returns `ArticleDraft` to workflow |
| `AuditWorkflow` | `Workflow` | Runs the score → guardrail → revise loop; halts at the ceiling. | `AuditEndpoint`, `SubmissionConsumer` | `ArticleEntity` |
| `ArticleEntity` | `EventSourcedEntity` | Holds the article lifecycle, every draft, every rubric score, and the final outcome. | `AuditWorkflow` | `ArticlesView` |
| `SubmissionQueue` | `EventSourcedEntity` | Logs each submitted article for replay and audit. | `AuditEndpoint`, `ArticleSimulator` | `SubmissionConsumer` |
| `ArticlesView` | `View` | List-of-articles read model. | `ArticleEntity` events | `AuditEndpoint` |
| `SubmissionConsumer` | `Consumer` | Subscribes to `SubmissionQueue` events; starts a workflow per submission. | `SubmissionQueue` events | `AuditWorkflow` |
| `ArticleSimulator` | `TimedAction` | Drips a sample article brief every 60 s from `sample-events/article-briefs.jsonl`. | scheduler | `SubmissionQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `ArticlesView`, records an `EvalRecorded` event for any round that completed since the last tick. | scheduler | `ArticleEntity` |
| `AuditEndpoint` | `HttpEndpoint` | `/api/articles/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `ArticlesView`, `SubmissionQueue`, `ArticleEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ArticleSubmission(String title, String bodyText, String targetKeyword, String submittedBy) {}

record ArticleDraft(String title, String bodyText, int wordCount, Instant draftedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record DimensionScore(String dimension, int score, String note) {}

record RubricFeedback(List<DimensionScore> failingDimensions, String improvementSummary) {}

record RubricScore(
    boolean passed,
    int compositeScore,
    List<DimensionScore> dimensions,
    RubricFeedback feedback,
    Instant scoredAt
) {}

record AuditRound(
    int roundNumber,
    ArticleDraft draft,
    GuardrailVerdict guardrail,
    Optional<RubricScore> rubricScore
) {}

record Article(
    String articleId,
    String title,
    String targetKeyword,
    int wordCountCeiling,
    int maxRounds,
    int acceptanceThreshold,
    ArticleStatus status,
    List<AuditRound> rounds,
    Optional<Integer> approvedAtRound,
    Optional<String> approvedBodyText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ArticleStatus { SCORING, REVISING, APPROVED, FAILED_FINAL }

enum RubricVerdict { PASS, FAIL }
```

### Events (on `ArticleEntity`)

`ArticleCreated`, `RoundDraftRecorded`, `RoundGuardrailVerdictRecorded`, `RoundScoreRecorded`, `ArticleApproved`, `ArticleFailedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/articles` — body `{ title, bodyText, targetKeyword, submittedBy? }` → `{ articleId }`. Starts a workflow.
- `GET /api/articles` — list all articles. Optional `?status=SCORING|REVISING|APPROVED|FAILED_FINAL`.
- `GET /api/articles/{id}` — one article (including every round and every rubric score).
- `GET /api/articles/sse` — server-sent events stream of every article change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "SEO Article Evaluation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, halt = red).
- **App UI** — form to submit an article, live list of articles with status pills, click-to-expand per-round timeline showing each draft, the guardrail verdict, the rubric verdict, and the failing dimension breakdown.

Browser title: `<title>Akka Sample: SEO Article Evaluation</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `OptimizerAgent`): a deterministic check that the revised draft's word count is at or below the per-article ceiling. Over-length drafts short-circuit back to the Optimizer with a structured feedback note (`reasonCode = OVER_WORD_CEILING`); they never reach the Rubric. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every round's rubric score is recorded as an `EvalRecorded` event with `{ roundNumber, verdict, compositeScore, wordCeilingExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow also emits an event on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-round timeline and in `/api/articles/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxRounds` without passing the threshold, the workflow ends with `ArticleFailedFinal`. The entity preserves every draft, every rubric score, the highest-scoring draft's text, and a structured rejection reason. The system never deletes drafts or terminates abruptly. Enforcement: system-level.

## 9. Agent prompts

- `RubricAgent` → `prompts/rubric-agent.md`. Scores a blog post against the 21-point helpful-content checklist; returns `RubricScore` with pass/fail and per-dimension breakdown.
- `OptimizerAgent` → `prompts/optimizer-agent.md`. Revises specific sections of an article in response to `RubricFeedback`; returns a revised `ArticleDraft`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit an article; it progresses `SCORING` → `REVISING` → `APPROVED` within the retry ceiling; the App UI shows every round's draft, guardrail verdict, and rubric score breakdown.
2. **J2 — halt at ceiling** — Submit an article whose rubric is impossible to satisfy (test mode forces `FAIL` every round); the article progresses through every round and lands in `FAILED_FINAL` with the highest-scoring draft preserved.
3. **J3 — guardrail block** — Submit an article with a low word-count ceiling; the Optimizer's first revision exceeds the ceiling; the guardrail short-circuits with `reasonCode = OVER_WORD_CEILING`, the Optimizer re-drafts under the ceiling, the loop continues.
4. **J4 — eval-event timeline** — The expanded view of any article shows one `EvalRecorded` event per scored round and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named seo-rubric-critic demonstrating the evaluator-optimizer ×
content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-content-editorial-seo-rubric-critic.
Java package io.akka.samples.seoarticleevaluation. Akka 3.6.0. HTTP port 9638.

Components to wire (exactly):
- 2 AutonomousAgents:
  * RubricAgent — definition() with
    capability(TaskAcceptance.of(SCORE_ARTICLE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/rubric-agent.md. Returns RubricScore{passed,
    compositeScore, dimensions, feedback, scoredAt} for the SCORE_ARTICLE task.
  * OptimizerAgent — definition() with
    capability(TaskAcceptance.of(REVISE_ARTICLE).maxIterationsPerTask(3)).
    System prompt from prompts/optimizer-agent.md. Returns ArticleDraft{title,
    bodyText, wordCount, draftedAt}. The REVISE_ARTICLE task takes
    (originalSubmission, priorDraft, rubricFeedback) as inputs.

- 1 Workflow AuditWorkflow with steps:
    startStep -> scoreStep -> [verdict PASS? approveStep : guardrailStep]
    guardrailStep -> [guardrail FAIL? reviseStep with feedback : scoreStep after revise]
    reviseStep -> guardrailStep -> scoreStep ->
    [verdict PASS? approveStep : (roundCount < maxRounds ?
       reviseStep with feedback : failStep)] -> END.
  scoreStep calls forAutonomousAgent(RubricAgent.class, articleId).runSingleTask(
    SCORE_ARTICLE) then forTask(taskId).result(SCORE_ARTICLE).
  reviseStep calls forAutonomousAgent(OptimizerAgent.class, articleId).runSingleTask(
    REVISE_ARTICLE). approveStep emits ArticleApproved.
  failStep emits ArticleFailedFinal with the highest-scoring draft's text as
    best-of and a structured rejectionReason ("max rounds reached (N)" or similar).
  Override settings() with stepTimeout(60s) on scoreStep and reviseStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    draft.wordCount() <= wordCountCeiling. On FAIL, emits
    RoundGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_WORD_CEILING", then transitions back to reviseStep with a
    structured feedback RubricFeedback("Draft exceeds the configured word-count
    ceiling; shorten and resubmit.").

- 1 EventSourcedEntity ArticleEntity holding state Article{articleId, title,
  targetKeyword, wordCountCeiling, maxRounds, acceptanceThreshold,
  ArticleStatus status, List<AuditRound> rounds,
  Optional<Integer> approvedAtRound, Optional<String> approvedBodyText,
  Optional<String> rejectionReason, Instant createdAt,
  Optional<Instant> finishedAt}.
  ArticleStatus enum: SCORING, REVISING, APPROVED, FAILED_FINAL.
  Events: ArticleCreated, RoundDraftRecorded, RoundGuardrailVerdictRecorded,
  RoundScoreRecorded, ArticleApproved, ArticleFailedFinal, EvalRecorded.
  Commands: createArticle, recordDraft, recordGuardrail, recordScore, approve,
  failFinal, recordEval, getArticle. emptyState() returns Article.initial("",
  "", "", 1500, 4, 75) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity SubmissionQueue with command enqueueArticle(title,
  bodyText, targetKeyword, submittedBy) emitting
  ArticleSubmitted{articleId, title, bodyText, targetKeyword, submittedBy,
  submittedAt}.

- 1 View ArticlesView with row type ArticleRow (mirrors Article; the rounds list
  is preserved as-is — the list is bounded at maxRounds so size stays
  reasonable). Table updater consumes ArticleEntity events. ONE query
  getAllArticles SELECT * AS articles FROM articles_view. No WHERE status
  filter — caller filters client-side because Akka cannot auto-index enum
  columns (Lesson 2).

- 1 Consumer SubmissionConsumer subscribed to SubmissionQueue events; on
  ArticleSubmitted starts an AuditWorkflow with the articleId as the
  workflow id.

- 2 TimedActions:
  * ArticleSimulator — every 60s, reads next line from
    src/main/resources/sample-events/article-briefs.jsonl and calls
    SubmissionQueue.enqueueArticle.
  * EvalSampler — every 30s, queries ArticlesView.getAllArticles, finds articles
    with a scored round that has not yet been recorded as an EvalRecorded event,
    and calls ArticleEntity.recordEval(roundNumber, verdict, compositeScore,
    wordCeilingExceeded). Idempotent per (articleId, roundNumber).

- 2 HttpEndpoints:
  * AuditEndpoint at /api with POST /articles, GET /articles,
    GET /articles/{id}, GET /articles/sse, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/. The POST
    /articles body is {title, bodyText, targetKeyword, submittedBy?}; missing
    submittedBy defaults to "anonymous". wordCountCeiling defaults to 1500,
    maxRounds defaults to 4, acceptanceThreshold defaults to 75.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- AuditTasks.java declaring two Task<R> constants: SCORE_ARTICLE (resultConformsTo
  RubricScore), REVISE_ARTICLE (ArticleDraft).
- Domain records ArticleSubmission, ArticleDraft, GuardrailVerdict, DimensionScore,
  RubricFeedback, RubricScore, AuditRound, Article; enums ArticleStatus, RubricVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9638
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from the
  canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  seo-rubric-critic.audit.max-rounds = 4,
  seo-rubric-critic.audit.default-word-count-ceiling = 1500,
  seo-rubric-critic.audit.acceptance-threshold = 75, overridable by env var.
- src/main/resources/sample-events/article-briefs.jsonl with 8 canned article
  brief lines, each shaped {"title":"...", "bodyText":"...", "targetKeyword":"...",
  "wordCountCeiling":1500}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = content-quality-evaluation,
  decisions.authority_level = editorial-recommendation,
  data.data_classes.pii = false, capabilities.content-generation = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/rubric-agent.md, prompts/optimizer-agent.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: SEO Article Evaluation",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-round timeline). Browser title exactly:
  <title>Akka Sample: SEO Article Evaluation</title>.

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
  named in Section 9: rubric-agent.json, optimizer-agent.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    rubric-agent.json — 6 RubricScore entries. Three return passed=true with
      compositeScore 76–92 and a full dimensions list with no failing rows.
      Two return passed=false with compositeScore 48–62 and a RubricFeedback
      payload naming 3–4 failing dimensions with improvement notes (e.g.,
      "E-E-A-T signals thin: add first-hand data", "keyword density 0.2% —
      below 0.5% target", "meta description absent"). One returns passed=false
      with compositeScore 35 used to exercise the halt in J2.
    optimizer-agent.json — 6 ArticleDraft entries. Three are improved drafts
      that tighten keyword placement and add data citations. Two are revision
      drafts at 900–1200 words responding to specific RubricFeedback. One is
      an intentionally over-ceiling draft (>1700 words) to exercise the
      guardrail in J3.
- A MockModelProvider.seedFor(articleId, roundNumber) helper makes the
  selection deterministic per (articleId, roundNumber) so the same article in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. RubricAgent
  and OptimizerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an AuditTasks companion declaring the two Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Article row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: AuditTasks.java is mandatory; generating RubricAgent or
  OptimizerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9638, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND theme variables for state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the reference
  does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements; removed panels are deleted from the HTML, not hidden with
  display:none.
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
