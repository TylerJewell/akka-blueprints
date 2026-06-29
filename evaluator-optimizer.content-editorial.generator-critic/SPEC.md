# SPEC — generator-critic

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Basic Reflection.
**One-line pitch:** Submit a topic; a generator agent drafts an essay; a reflector agent scores it against coherence and clarity criteria; the two iterate until the reflector accepts or the loop hits its round ceiling; the accepted draft is released only after a policy guardrail approves it.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a reviewer agent (`ReflectorAgent`), feeding each critique back into the next draft until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every round's reflector verdict for downstream quality measurement, and a **before-agent-response guardrail** that gates the final accepted draft against a content-policy rule before it is released to the caller.

## 3. User-facing flows

The user opens the App UI tab and submits a topic (free text) along with an optional word-count ceiling.

1. The system creates an `Essay` record in `DRAFTING` and starts a `ReflectionWorkflow`.
2. The Generator drafts round #1: a coherent short essay on the topic within the word-count ceiling.
3. The Reflector scores the draft against a fixed rubric (coherence, clarity, topic adherence, factual plausibility) and returns either `ACCEPT` with a one-line rationale, or `REVISE` with a typed `ReflectionNotes` payload (up to three bullets).
4. On `ACCEPT`, the workflow runs the output guardrail on the accepted draft. If the draft passes the policy check, the essay transitions to `RELEASED`. If it fails, the Generator is asked to produce one policy-clean revision; that revision bypasses the Reflector and goes straight to the guardrail again.
5. On `REVISE`, the workflow records the round, the reflection, and the reflector's verdict on the entity, then calls the Generator again with the reflection attached. The Generator produces round #2.
6. If the loop reaches `maxRounds` (default 4) without an `ACCEPT`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the best-scoring round's draft is preserved on the entity along with every reflection for audit, and a `ReflectionRecorded` event captures the rejection.

A `TopicSimulator` (TimedAction) drips a canned topic every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Drafts an essay on the topic; on a revision call, accepts prior reflection notes and produces an improved draft. | `ReflectionWorkflow` | returns `EssayDraft` to workflow |
| `ReflectorAgent` | `AutonomousAgent` | Scores a draft against the rubric; returns `ACCEPT` or `REVISE` with structured notes. | `ReflectionWorkflow` | returns `Reflection` to workflow |
| `ReflectionWorkflow` | `Workflow` | Runs the generate → reflect → revise loop; applies the output guardrail on acceptance; halts at the ceiling. | `EssayEndpoint`, `SubmissionConsumer` | `EssayEntity` |
| `EssayEntity` | `EventSourcedEntity` | Holds the essay lifecycle, every round's draft, every reflection, and the final outcome. | `ReflectionWorkflow` | `EssaysView` |
| `SubmissionQueue` | `EventSourcedEntity` | Logs each submitted topic for replay and audit. | `EssayEndpoint`, `TopicSimulator` | `SubmissionConsumer` |
| `EssaysView` | `View` | List-of-essays read model. | `EssayEntity` events | `EssayEndpoint` |
| `SubmissionConsumer` | `Consumer` | Subscribes to `SubmissionQueue` events; starts a workflow per submission. | `SubmissionQueue` events | `ReflectionWorkflow` |
| `TopicSimulator` | `TimedAction` | Drips a sample topic every 60 s from `sample-events/essay-topics.jsonl`. | scheduler | `SubmissionQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `EssaysView`, records a `ReflectionRecorded` event for any round that completed since the last tick. | scheduler | `EssayEntity` |
| `EssayEndpoint` | `HttpEndpoint` | `/api/essays/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `EssaysView`, `SubmissionQueue`, `EssayEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TopicRequest(String topic, int wordCeiling, String requestedBy) {}

record EssayDraft(String text, int wordCount, Instant draftedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ReflectionNotes(List<String> bullets, String overallRationale) {}

record Reflection(ReflectorVerdict verdict, ReflectionNotes notes, int score, Instant evaluatedAt) {}

record Round(
    int roundNumber,
    EssayDraft draft,
    GuardrailVerdict guardrail,
    Optional<Reflection> reflection
) {}

record Essay(
    String essayId,
    String topic,
    int wordCeiling,
    int maxRounds,
    EssayStatus status,
    List<Round> rounds,
    Optional<Integer> acceptedRoundNumber,
    Optional<String> releasedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EssayStatus { DRAFTING, REFLECTING, ACCEPTED, RELEASED, REJECTED_FINAL }

enum ReflectorVerdict { ACCEPT, REVISE }
```

### Events (on `EssayEntity`)

`EssayCreated`, `RoundDrafted`, `RoundGuardrailVerdictRecorded`, `RoundReflected`, `EssayAccepted`, `EssayReleased`, `EssayRejectedFinal`, `ReflectionRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/essays` — body `{ topic, wordCeiling?, requestedBy? }` → `{ essayId }`. Starts a workflow.
- `GET /api/essays` — list all essays. Optional `?status=DRAFTING|REFLECTING|ACCEPTED|RELEASED|REJECTED_FINAL`.
- `GET /api/essays/{id}` — one essay (including every round and every reflection).
- `GET /api/essays/sse` — server-sent events stream of every essay change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Basic Reflection"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a topic, live list of essays with status pills, click-to-expand per-round timeline showing each draft, the guardrail verdict, the reflector's verdict, and the reflector's notes.

Browser title: `<title>Akka Sample: Basic Reflection</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `GeneratorAgent`): applied to the accepted draft before it is released to the caller. Checks that the text does not contain any flagged phrases from a short built-in policy list (empty by default; operators extend it via `application.conf`). A failing draft is sent back to the Generator with a structured feedback note (`reasonCode = POLICY_VIOLATION`); the revised draft bypasses the Reflector and goes directly back to the guardrail. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every round's reflection is recorded as a `ReflectionRecorded` event with `{ roundNumber, verdict, score, guardrailPassed }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits a `ReflectionRecorded` event on terminal transitions. Enforcement: non-blocking.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Drafts an essay on the topic within the word ceiling; on a revision call, takes the prior `ReflectionNotes` as input and produces an improved draft.
- `ReflectorAgent` → `prompts/reflector.md`. Scores a draft against the fixed rubric; returns `ACCEPT` with a one-line rationale or `REVISE` with up to three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a topic; essay progresses `DRAFTING` → `REFLECTING` → `ACCEPTED` → `RELEASED` within the round ceiling; the App UI shows every round's draft and reflection.
2. **J2 — halt at ceiling** — Submit a topic whose rubric is impossible (test mode forces the Reflector to `REVISE` every round); essay progresses through every round and lands in `REJECTED_FINAL` with the best draft preserved and a structured rejection reason.
3. **J3 — guardrail block on release** — Submit a topic that causes the Generator to include a flagged phrase; the guardrail blocks the draft pre-release, the Generator revises, the clean draft is released.
4. **J4 — eval-event timeline** — The expanded view of any essay shows one `ReflectionRecorded` event per round and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named generator-critic demonstrating the evaluator-optimizer ×
content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-content-editorial-generator-critic.
Java package io.akka.samples.basicreflection. Akka 3.6.0. HTTP port 9732.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(DRAFT_ESSAY).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ESSAY).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns EssayDraft{text,
    wordCount, draftedAt} for both DRAFT_ESSAY and REVISE_ESSAY. The
    REVISE_ESSAY task takes (originalTopic, priorDraft, ReflectionNotes) as
    inputs. A third task POLICY_REVISE also returns EssayDraft; it takes
    (originalTopic, priorDraft, policyFeedback: String) as inputs.
  * ReflectorAgent — definition() with
    capability(TaskAcceptance.of(REFLECT).maxIterationsPerTask(2)). System
    prompt from prompts/reflector.md. Returns Reflection{verdict, notes, score,
    evaluatedAt} where verdict is the ReflectorVerdict enum (ACCEPT | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow ReflectionWorkflow with steps:
    startStep -> draftStep -> reflectStep ->
    [verdict ACCEPT? guardrailStep : (roundCount < maxRounds ?
       draftStep with reflection attached : rejectStep)] ->
    guardrailStep -> [guardrail passed? releaseStep : policyReviseStep ->
      guardrailStep] -> END.
  draftStep calls forAutonomousAgent(GeneratorAgent.class, essayId).runSingleTask(
    DRAFT_ESSAY or REVISE_ESSAY) then forTask(taskId).result(DRAFT_ESSAY or
    REVISE_ESSAY). reflectStep calls forAutonomousAgent(ReflectorAgent.class,
    essayId).runSingleTask(REFLECT). guardrailStep is a pure-function step: checks
    the draft text against the operator-configured policy phrase list in
    basic-reflection.policy.forbidden-phrases (default empty list). On
    PASS, emits RoundGuardrailVerdictRecorded with passed=true. On FAIL, emits
    RoundGuardrailVerdictRecorded with passed=false and reasonCode=POLICY_VIOLATION,
    then transitions to policyReviseStep. policyReviseStep calls
    GeneratorAgent.POLICY_REVISE with a structured policyFeedback string listing
    the offending phrases. releaseStep emits EssayReleased with the final text.
    rejectStep emits EssayRejectedFinal with the highest-scoring round's text as
    best-of and a structured rejectionReason.
    Override settings() with stepTimeout(60s) on draftStep and reflectStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).

- 1 EventSourcedEntity EssayEntity holding state Essay{essayId, topic,
  wordCeiling, maxRounds, EssayStatus status, List<Round> rounds,
  Optional<Integer> acceptedRoundNumber, Optional<String> releasedText,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. EssayStatus enum: DRAFTING, REFLECTING, ACCEPTED, RELEASED,
  REJECTED_FINAL. Events: EssayCreated, RoundDrafted,
  RoundGuardrailVerdictRecorded, RoundReflected, EssayAccepted, EssayReleased,
  EssayRejectedFinal, ReflectionRecorded. Commands: createEssay, recordDraft,
  recordGuardrail, recordReflection, accept, release, rejectFinal,
  recordReflectionEval, getEssay. emptyState() returns Essay.initial("", "", 400, 4)
  with no commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity SubmissionQueue with command submitTopic(topic,
  wordCeiling, requestedBy) emitting TopicSubmitted{essayId, topic,
  wordCeiling, requestedBy, submittedAt}.

- 1 View EssaysView with row type EssayRow (mirrors Essay; the rounds list
  is preserved as-is — the list is bounded at maxRounds so size stays
  reasonable). Table updater consumes EssayEntity events. ONE query
  getAllEssays SELECT * AS essays FROM essays_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer SubmissionConsumer subscribed to SubmissionQueue events; on
  TopicSubmitted starts a ReflectionWorkflow with the essayId as the
  workflow id.

- 2 TimedActions:
  * TopicSimulator — every 60s, reads next line from
    src/main/resources/sample-events/essay-topics.jsonl and calls
    SubmissionQueue.submitTopic.
  * EvalSampler — every 30s, queries EssaysView.getAllEssays, finds essays
    with a reflected round that has not yet been recorded as a
    ReflectionRecorded event, and calls EssayEntity.recordReflectionEval(
    roundNumber, verdict, score, guardrailPassed). Idempotent per (essayId, roundNumber).

- 2 HttpEndpoints:
  * EssayEndpoint at /api with POST /essays, GET /essays, GET /essays/{id},
    GET /essays/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /essays body
    is {topic, wordCeiling?, requestedBy?}; missing wordCeiling defaults to
    400, missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- EssayTasks.java declaring four Task<R> constants: DRAFT_ESSAY (resultConformsTo
  EssayDraft), REVISE_ESSAY (EssayDraft), POLICY_REVISE (EssayDraft), REFLECT (Reflection).
- Domain records EssayDraft, GuardrailVerdict, ReflectionNotes, Reflection,
  Round, Essay; enums EssayStatus, ReflectorVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9732 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  basic-reflection.reflection.max-rounds = 4 and
  basic-reflection.reflection.default-word-ceiling = 400 and
  basic-reflection.policy.forbidden-phrases = [], all overridable by env var.
- src/main/resources/sample-events/essay-topics.jsonl with 8 canned topic
  lines, each shaped {"topic":"...", "wordCeiling":400}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = content-generation-with-review,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.content-generation = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/reflector.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Basic Reflection",
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
  per-round timeline). Browser title exactly:
  <title>Akka Sample: Basic Reflection</title>.

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
  named in Section 9: generator.json, reflector.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 EssayDraft entries. Three are first-pass essays
      between 300 and 390 words on topics from essay-topics.jsonl.
      Two are revision drafts that address prior reflection feedback
      (tighter, more focused). One is an intentionally policy-violating
      draft containing a placeholder forbidden phrase for J3 testing.
    reflector.json — 6 Reflection entries. Three return verdict=ACCEPT with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a ReflectionNotes payload of
      three bullets ("opening paragraph lacks a clear thesis", "paragraph 3
      introduces an unsupported claim", "conclusion does not follow from
      the body").
- A MockModelProvider.seedFor(essayId, roundNumber) helper makes the
  selection deterministic per (essayId, roundNumber) so the same essay in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeneratorAgent
  and ReflectorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an EssayTasks companion declaring the four Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Essay row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: EssayTasks.java is mandatory; generating GeneratorAgent or
  ReflectorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9732, declared in application.conf
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
