# SPEC — self-eval-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Eval Loop Posts.
**One-line pitch:** Type a brief; a poet agent drafts a Shakespearean-style post; a critic agent scores it against length and style rules; the two iterate until the critic accepts or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`PoetAgent`) and a reviewer agent (`CriticAgent`), feeding each critique back into the next draft until convergence or a halt. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, an **output guardrail** that gates each draft against a deterministic length rule before the critic runs, and a **halt** that ends the loop gracefully at the retry ceiling without leaving the post in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a brief (a topic plus a target character ceiling).

1. The system creates a `Post` record in `DRAFTING` and starts a `RefinementWorkflow`.
2. The Poet drafts attempt #1: a short Shakespearean-style passage on the brief.
3. The output guardrail vets the draft against the character ceiling. Over-length drafts are short-circuited back to the Poet with a deterministic feedback note; they never reach the Critic.
4. The Critic scores the draft against a fixed rubric (length, register, imagery, period vocabulary) and returns either `ACCEPT` with a one-line rationale, or `REVISE` with a typed `CritiqueNotes` payload (three bullets at most).
5. On `ACCEPT`, the workflow transitions the post to `ACCEPTED` with the winning attempt's text and the critic's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the critique, and the critic's verdict on the entity, then calls the Poet again with the critique attached. The Poet produces attempt #2.
7. If the loop reaches `maxAttempts` (default 4) without an `ACCEPT`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring attempt is preserved on the entity along with every critique for audit, and an `EvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PoetAgent` | `AutonomousAgent` | Drafts a Shakespearean-style post on a brief; accepts prior critique on revisions. | `RefinementWorkflow` | returns `DraftPassage` to workflow |
| `CriticAgent` | `AutonomousAgent` | Scores a draft against the rubric; returns `ACCEPT` or `REVISE` with notes. | `RefinementWorkflow` | returns `Critique` to workflow |
| `RefinementWorkflow` | `Workflow` | Runs the draft → guardrail → critique → revise loop; halts at the ceiling. | `PoetryEndpoint`, `DraftRequestConsumer` | `PostEntity` |
| `PostEntity` | `EventSourcedEntity` | Holds the post lifecycle, every attempt, every critique, and the final outcome. | `RefinementWorkflow` | `PostsView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted brief for replay and audit. | `PoetryEndpoint`, `RequestSimulator` | `DraftRequestConsumer` |
| `PostsView` | `View` | List-of-posts read model. | `PostEntity` events | `PoetryEndpoint` |
| `DraftRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `RefinementWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample brief every 60 s from `sample-events/poetry-briefs.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `PostsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `PostEntity` |
| `PoetryEndpoint` | `HttpEndpoint` | `/api/posts/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `PostsView`, `RequestQueue`, `PostEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Brief(String topic, int characterCeiling, String requestedBy) {}

record DraftPassage(String text, int characterCount, Instant draftedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record CritiqueNotes(List<String> bullets, String overallRationale) {}

record Critique(CriticVerdict verdict, CritiqueNotes notes, int score, Instant evaluatedAt) {}

record Attempt(
    int attemptNumber,
    DraftPassage draft,
    GuardrailVerdict guardrail,
    Optional<Critique> critique
) {}

record Post(
    String postId,
    String topic,
    int characterCeiling,
    int maxAttempts,
    PostStatus status,
    List<Attempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PostStatus { DRAFTING, EVALUATING, ACCEPTED, REJECTED_FINAL }

enum CriticVerdict { ACCEPT, REVISE }
```

### Events (on `PostEntity`)

`PostCreated`, `AttemptDrafted`, `AttemptGuardrailVerdictRecorded`, `AttemptCritiqued`, `PostAccepted`, `PostRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/posts` — body `{ topic, characterCeiling?, requestedBy? }` → `{ postId }`. Starts a workflow.
- `GET /api/posts` — list all posts. Optional `?status=DRAFTING|EVALUATING|ACCEPTED|REJECTED_FINAL`.
- `GET /api/posts/{id}` — one post (including every attempt and every critique).
- `GET /api/posts/sse` — server-sent events stream of every post change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Self-Eval Loop Posts"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, halt = red).
- **App UI** — form to submit a brief, live list of posts with status pills, click-to-expand per-attempt timeline showing each draft, the guardrail verdict, the critic's verdict, and the critic's notes.

Browser title: `<title>Akka Sample: Self-Eval Loop Posts</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `PoetAgent`): a deterministic check that the draft's character count is at or below the per-post ceiling. Over-length drafts short-circuit back to the Poet with a structured feedback note (`reasonCode = OVER_CEILING`); they never reach the Critic. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's critique is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, ceilingExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/posts/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without an `ACCEPT`, the workflow ends with `PostRejectedFinal`. The entity preserves every attempt, every critique, the best-scoring attempt's text, and a structured rejection reason. The system never deletes drafts or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `PoetAgent` → `prompts/poet.md`. Drafts a Shakespearean-style passage on the brief; on a revision call, takes the prior `CritiqueNotes` as input and produces a new draft.
- `CriticAgent` → `prompts/critic.md`. Scores a draft against the fixed rubric; returns `ACCEPT` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a brief; post progresses `DRAFTING` → `EVALUATING` → `ACCEPTED` within the retry ceiling; the App UI shows every attempt's draft and critique.
2. **J2 — halt at ceiling** — Submit a brief whose rubric is impossible (test mode forces the Critic to `REVISE` every attempt); post progresses through every attempt and lands in `REJECTED_FINAL` with the best draft preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a brief with `characterCeiling = 100`; the Poet's first draft exceeds the ceiling; the guardrail short-circuits with `reasonCode = OVER_CEILING`, the Poet re-drafts under the ceiling, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any post shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-eval-loop demonstrating the evaluator-optimizer ×
content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id self-eval-loop. Java package
io.akka.samples.selfevalloop. Akka 3.6.0. HTTP port 9346.

Components to wire (exactly):
- 2 AutonomousAgents:
  * PoetAgent — definition() with
    capability(TaskAcceptance.of(DRAFT).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_DRAFT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/poet.md. Returns DraftPassage{text,
    characterCount, draftedAt} for both DRAFT and REVISE_DRAFT. The
    REVISE_DRAFT task takes (originalBrief, priorDraft, CritiqueNotes) as
    inputs.
  * CriticAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)). System
    prompt from prompts/critic.md. Returns Critique{verdict, notes, score,
    evaluatedAt} where verdict is the CriticVerdict enum (ACCEPT | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow RefinementWorkflow with steps:
    startStep -> draftStep -> guardrailStep -> [guardrail FAIL? draftStep
    again with structured feedback : critiqueStep] ->
    [verdict ACCEPT? acceptStep : (attemptCount < maxAttempts ?
       draftStep with critique attached : rejectStep)] -> END.
  draftStep calls forAutonomousAgent(PoetAgent.class, postId).runSingleTask(
    DRAFT or REVISE_DRAFT) then forTask(taskId).result(DRAFT or
    REVISE_DRAFT). critiqueStep calls forAutonomousAgent(CriticAgent.class,
    postId).runSingleTask(EVALUATE). acceptStep emits PostAccepted.
    rejectStep emits PostRejectedFinal with the highest-scoring attempt's
    text as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on draftStep and critiqueStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    draft.characterCount() <= ceiling. On FAIL, emits
    AttemptGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_CEILING", then transitions back to draftStep with a
    structured feedback CritiqueNotes("Draft exceeds the configured
    character ceiling; trim and resubmit.").

- 1 EventSourcedEntity PostEntity holding state Post{postId, topic,
  characterCeiling, maxAttempts, PostStatus status, List<Attempt> attempts,
  Optional<Integer> acceptedAttemptNumber, Optional<String> acceptedText,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. PostStatus enum: DRAFTING, EVALUATING, ACCEPTED,
  REJECTED_FINAL. Events: PostCreated, AttemptDrafted,
  AttemptGuardrailVerdictRecorded, AttemptCritiqued, PostAccepted,
  PostRejectedFinal, EvalRecorded. Commands: createPost, recordDraft,
  recordGuardrail, recordCritique, accept, rejectFinal, recordEval,
  getPost. emptyState() returns Post.initial("", "", 280, 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueBrief(topic,
  characterCeiling, requestedBy) emitting BriefSubmitted{postId, topic,
  characterCeiling, requestedBy, submittedAt}.

- 1 View PostsView with row type PostRow (mirrors Post; the attempts list
  is preserved as-is — the list is bounded at maxAttempts so size stays
  reasonable). Table updater consumes PostEntity events. ONE query
  getAllPosts SELECT * AS posts FROM posts_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer DraftRequestConsumer subscribed to RequestQueue events; on
  BriefSubmitted starts a RefinementWorkflow with the postId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/poetry-briefs.jsonl and calls
    RequestQueue.enqueueBrief.
  * EvalSampler — every 30s, queries PostsView.getAllPosts, finds posts
    with a critiqued attempt that has not yet been recorded as an
    EvalRecorded event, and calls PostEntity.recordEval(attemptNumber,
    verdict, score, ceilingExceeded). Idempotent per (postId, attemptNumber).

- 2 HttpEndpoints:
  * PoetryEndpoint at /api with POST /posts, GET /posts, GET /posts/{id},
    GET /posts/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /posts body
    is {topic, characterCeiling?, requestedBy?}; missing characterCeiling
    defaults to 280, missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PoetryTasks.java declaring three Task<R> constants: DRAFT (resultConformsTo
  DraftPassage), REVISE_DRAFT (DraftPassage), EVALUATE (Critique).
- Domain records DraftPassage, GuardrailVerdict, CritiqueNotes, Critique,
  Attempt, Post; enums PostStatus, CriticVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9346 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  self-eval-loop.refinement.max-attempts = 4 and
  self-eval-loop.refinement.default-character-ceiling = 280, overridable by
  env var.
- src/main/resources/sample-events/poetry-briefs.jsonl with 8 canned brief
  lines, each shaped {"topic":"...", "characterCeiling":280}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = creative-writing-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/poet.md, prompts/critic.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Self-Eval Loop Posts",
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
  <title>Akka Sample: Self-Eval Loop Posts</title>.

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
  named in Section 9: poet.json, critic.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    poet.json — 6 DraftPassage entries. Three are "first-pass" passages
      between 200 and 270 characters in a Shakespearean register on the
      topics in poetry-briefs.jsonl. Two are "revision" passages that
      reference the prior critique (shorter, sharper). One is an
      intentionally over-ceiling passage (>310 characters) used to
      exercise the guardrail in J3.
    critic.json — 6 Critique entries. Three return verdict=ACCEPT with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a CritiqueNotes payload of
      three bullets ("metre is uneven in line 2", "Elizabethan
      vocabulary thins after line 4", "imagery is contemporary").
- A MockModelProvider.seedFor(postId, attemptNumber) helper makes the
  selection deterministic per (postId, attemptNumber) so the same post in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. PoetAgent
  and CriticAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a PoetryTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Post row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: PoetryTasks.java is mandatory; generating PoetAgent or
  CriticAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9346, declared in application.conf
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
