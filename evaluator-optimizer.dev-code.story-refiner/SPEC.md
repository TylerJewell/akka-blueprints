# SPEC — story-refiner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** SDLC User Story Refiner.
**One-line pitch:** Submit a feature brief; a drafter agent writes a structured user story; a reviewer agent scores it against completeness, testability, and scope rules; the two iterate until the reviewer approves or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`StoryDrafterAgent`) and a reviewer agent (`ReviewerAgent`), feeding each review's structured notes back into the next draft until convergence or a halt. The blueprint also demonstrates one governance mechanism — an **eval-event** that records every cycle's verdict for downstream quality measurement. Unlike content-generation loops, this domain produces structured output (role, goal, benefit, ACs) where each field is independently scoreable, making per-field scoring feedback actionable across attempts.

## 3. User-facing flows

The user opens the App UI tab and submits a brief (a feature description plus a target team and optional story-type hint).

1. The system creates a `Story` record in `DRAFTING` and starts a `RefinementWorkflow`.
2. The Drafter writes attempt #1: a user story with a role, goal, benefit, and 2–5 acceptance criteria.
3. The Reviewer scores the draft against a fixed rubric (role clarity, goal specificity, benefit value, AC testability, scope tightness) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `ReviewNotes` payload (up to four bullets).
4. On `APPROVE`, the workflow transitions the story to `APPROVED` with the winning attempt's content and the reviewer's rationale.
5. On `REVISE`, the workflow records the attempt, the review, and the reviewer's verdict on the entity, then calls the Drafter again with the notes attached. The Drafter produces attempt #2.
6. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring attempt is preserved on the entity along with every review for audit, and an `EvalRecorded` event captures the rejection.

A `RequestSimulator` (TimedAction) drips a canned brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StoryDrafterAgent` | `AutonomousAgent` | Drafts a structured user story from a brief; incorporates prior reviewer notes on revisions. | `RefinementWorkflow` | returns `StoryDraft` to workflow |
| `ReviewerAgent` | `AutonomousAgent` | Scores a draft against the rubric; returns `APPROVE` or `REVISE` with notes. | `RefinementWorkflow` | returns `Review` to workflow |
| `RefinementWorkflow` | `Workflow` | Runs the draft → review → revise loop; halts at the ceiling. | `StoryEndpoint`, `StoryRequestConsumer` | `StoryEntity` |
| `StoryEntity` | `EventSourcedEntity` | Holds the story lifecycle, every attempt, every review, and the final outcome. | `RefinementWorkflow` | `StoriesView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted brief for replay and audit. | `StoryEndpoint`, `RequestSimulator` | `StoryRequestConsumer` |
| `StoriesView` | `View` | List-of-stories read model. | `StoryEntity` events | `StoryEndpoint` |
| `StoryRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a workflow per submission. | `RequestQueue` events | `RefinementWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample brief every 60 s from `sample-events/story-briefs.jsonl`. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `StoriesView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `StoryEntity` |
| `StoryEndpoint` | `HttpEndpoint` | `/api/stories/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `StoriesView`, `RequestQueue`, `StoryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record StoryBrief(String featureDescription, String targetTeam, Optional<String> storyTypeHint, String requestedBy) {}

record StoryDraft(
    String role,
    String goal,
    String benefit,
    List<String> acceptanceCriteria,
    int fieldCount,
    Instant draftedAt
) {}

record ReviewNotes(List<String> bullets, String overallRationale) {}

record Review(ReviewVerdict verdict, ReviewNotes notes, int score, Instant reviewedAt) {}

record Attempt(
    int attemptNumber,
    StoryDraft draft,
    Optional<Review> review
) {}

record Story(
    String storyId,
    String featureDescription,
    String targetTeam,
    int maxAttempts,
    StoryStatus status,
    List<Attempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<StoryDraft> approvedDraft,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum StoryStatus { DRAFTING, REVIEWING, APPROVED, REJECTED_FINAL }

enum ReviewVerdict { APPROVE, REVISE }
```

### Events (on `StoryEntity`)

`StoryCreated`, `AttemptDrafted`, `AttemptReviewed`, `StoryApproved`, `StoryRejectedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/stories` — body `{ featureDescription, targetTeam?, storyTypeHint?, requestedBy? }` → `{ storyId }`. Starts a workflow.
- `GET /api/stories` — list all stories. Optional `?status=DRAFTING|REVIEWING|APPROVED|REJECTED_FINAL`.
- `GET /api/stories/{id}` — one story (including every attempt and every review).
- `GET /api/stories/sse` — server-sent events stream of every story change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "SDLC User Story Refiner"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue).
- **App UI** — form to submit a feature brief, live list of stories with status pills, click-to-expand per-attempt timeline showing each draft, the reviewer's verdict, and the reviewer's notes.

Browser title: `<title>Akka Sample: SDLC User Story Refiner</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every cycle's review is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, score, fieldScores }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/stories/{id}`.

## 9. Agent prompts

- `StoryDrafterAgent` → `prompts/story-drafter.md`. Drafts a structured user story from a feature description; on a revision call, takes the prior `ReviewNotes` as input and produces a revised draft.
- `ReviewerAgent` → `prompts/reviewer.md`. Scores a draft against the fixed rubric; returns `APPROVE` with a one-line rationale or `REVISE` with up to four short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a brief; story progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's draft fields and review.
2. **J2 — halt at ceiling** — Submit a brief whose rubric is impossible to satisfy (test mode forces the Reviewer to `REVISE` every attempt); story lands in `REJECTED_FINAL` with the best draft preserved and a structured rejection reason.
3. **J3 — revision incorporation** — Reviewer returns `REVISE` with a specific bullet ("acceptance criteria are not independently testable"); the Drafter's next attempt addresses that bullet; the Reviewer's score on that dimension improves.
4. **J4 — eval-event timeline** — The expanded view of any story shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named story-refiner demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-story-refiner.
Java package io.akka.samples.sdlcuserstoryrefiner. Akka 3.6.0. HTTP port 9223.

Components to wire (exactly):
- 2 AutonomousAgents:
  * StoryDrafterAgent — definition() with
    capability(TaskAcceptance.of(DRAFT).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_DRAFT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/story-drafter.md. Returns StoryDraft{role,
    goal, benefit, acceptanceCriteria, fieldCount, draftedAt} for both DRAFT
    and REVISE_DRAFT. The REVISE_DRAFT task takes (originalBrief, priorDraft,
    ReviewNotes) as inputs.
  * ReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/reviewer.md. Returns Review{verdict, notes, score,
    reviewedAt} where verdict is the ReviewVerdict enum (APPROVE | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow RefinementWorkflow with steps:
    startStep -> draftStep -> reviewStep ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       draftStep with review notes attached : rejectStep)] -> END.
  draftStep calls forAutonomousAgent(StoryDrafterAgent.class, storyId).runSingleTask(
    DRAFT or REVISE_DRAFT) then forTask(taskId).result(DRAFT or REVISE_DRAFT).
  reviewStep calls forAutonomousAgent(ReviewerAgent.class, storyId).runSingleTask(REVIEW).
  approveStep emits StoryApproved. rejectStep emits StoryRejectedFinal with the
  highest-scoring attempt's draft as best-of and a structured rejectionReason.
  Override settings() with stepTimeout(60s) on draftStep and reviewStep, and
  defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).

- 1 EventSourcedEntity StoryEntity holding state Story{storyId, featureDescription,
  targetTeam, maxAttempts, StoryStatus status, List<Attempt> attempts,
  Optional<Integer> approvedAttemptNumber, Optional<StoryDraft> approvedDraft,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. StoryStatus enum: DRAFTING, REVIEWING, APPROVED, REJECTED_FINAL.
  Events: StoryCreated, AttemptDrafted, AttemptReviewed, StoryApproved,
  StoryRejectedFinal, EvalRecorded. Commands: createStory, recordDraft,
  recordReview, approve, rejectFinal, recordEval, getStory. emptyState()
  returns Story.initial("", "", "", 4) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity RequestQueue with command enqueueBrief(featureDescription,
  targetTeam, storyTypeHint, requestedBy) emitting BriefSubmitted{storyId,
  featureDescription, targetTeam, storyTypeHint, requestedBy, submittedAt}.

- 1 View StoriesView with row type StoryRow (mirrors Story; attempts list is
  bounded at maxAttempts). Table updater consumes StoryEntity events. ONE
  query getAllStories SELECT * AS stories FROM stories_view. No WHERE status
  filter — caller filters client-side because Akka cannot auto-index enum
  columns (Lesson 2).

- 1 Consumer StoryRequestConsumer subscribed to RequestQueue events; on
  BriefSubmitted starts a RefinementWorkflow with the storyId as the
  workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/story-briefs.jsonl and calls
    RequestQueue.enqueueBrief.
  * EvalSampler — every 30s, queries StoriesView.getAllStories, finds stories
    with a reviewed attempt that has not yet been recorded as an EvalRecorded
    event, and calls StoryEntity.recordEval(attemptNumber, verdict, score,
    fieldScores). Idempotent per (storyId, attemptNumber).

- 2 HttpEndpoints:
  * StoryEndpoint at /api with POST /stories, GET /stories, GET /stories/{id},
    GET /stories/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /stories body
    is {featureDescription, targetTeam?, storyTypeHint?, requestedBy?};
    missing targetTeam defaults to "unspecified", missing requestedBy defaults
    to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- StoryTasks.java declaring three Task<R> constants: DRAFT (resultConformsTo
  StoryDraft), REVISE_DRAFT (StoryDraft), REVIEW (Review).
- Domain records StoryDraft, ReviewNotes, Review, Attempt, Story; enums
  StoryStatus, ReviewVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9223 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  story-refiner.refinement.max-attempts = 4 and
  story-refiner.refinement.default-team = "unspecified", overridable by
  env var.
- src/main/resources/sample-events/story-briefs.jsonl with 8 canned brief
  lines, each shaped {"featureDescription":"...", "targetTeam":"platform"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control (E1 eval-event
  on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = user-story-refinement,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/story-drafter.md, prompts/reviewer.md loaded at agent startup as
  system prompts.
- README.md at the project root: title "Akka Sample: SDLC User Story Refiner",
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
  <title>Akka Sample: SDLC User Story Refiner</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
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
  named in Section 9: story-drafter.json, reviewer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    story-drafter.json — 6 StoryDraft entries. Three are "first-pass" drafts
      with well-formed role/goal/benefit and 2-4 acceptance criteria on topics
      in story-briefs.jsonl. Two are "revision" drafts that demonstrably
      tighten scope or improve AC testability in response to a prior review.
      One has vague acceptance criteria that the reviewer will REVISE.
    reviewer.json — 6 Review entries. Three return verdict=APPROVE with
      score=4 or 5 and a one-sentence rationale. Three return verdict=REVISE
      with score=2 or 3 and a ReviewNotes payload of up to four bullets
      (e.g., "acceptance criteria are not independently testable",
      "goal conflates two distinct user needs", "benefit is not quantified",
      "role is too broad — specify the persona").
- A MockModelProvider.seedFor(storyId, attemptNumber) helper makes the
  selection deterministic per (storyId, attemptNumber) so the same story
  in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. StoryDrafterAgent
  and ReviewerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a StoryTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Story row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: StoryTasks.java is mandatory; generating StoryDrafterAgent or
  ReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9223, declared in application.conf
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
