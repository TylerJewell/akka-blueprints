# SPEC — image-policy-scorer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Image Policy Scorer.
**One-line pitch:** Submit a text prompt; a generator agent produces an image description; a policy scorer agent evaluates it against brand-safety, content-appropriateness, and platform-policy rules; the two iterate until the scorer approves or the loop reaches its attempt ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a reviewer agent (`ScorerAgent`), feeding each policy failure back into the next generation until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **after-llm-response guardrail** that gates each image description against a deterministic safety rule before the scorer runs, and an **on-decision eval-event** that records every cycle's verdict for downstream quality and compliance measurement. An implicit halt at the retry ceiling ends the loop gracefully and preserves every attempt for audit.

## 3. User-facing flows

The user opens the App UI tab and submits a prompt (a description of the desired image plus an optional audience-tier label).

1. The system creates an `Image` record in `GENERATING` and starts a `ScoringWorkflow`.
2. The Generator produces attempt #1: a structured image description with a content-category tag and a brand-safety signal.
3. The safety gate vets the description against the prohibited-signal list. Descriptions that match a prohibited signal are short-circuited back to the Generator with a deterministic feedback note; they never reach the Scorer.
4. The Scorer evaluates the description against a multi-dimension rubric (brand safety, content appropriateness, platform policy, audience fit) and returns either `PASS` with a one-line rationale, or `FAIL` with a typed `PolicyNotes` payload (up to three bullets).
5. On `PASS`, the workflow transitions the image to `APPROVED` with the winning attempt's description and the scorer's rationale.
6. On `FAIL`, the workflow records the attempt, the gate verdict, the policy notes, and the scorer's verdict on the entity, then calls the Generator again with the notes attached.
7. If the loop reaches `maxAttempts` (default 4) without a `PASS`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the highest-scoring attempt is preserved on the entity along with every policy note for audit, and a `PolicyEvalRecorded` event captures the rejection.

A `PromptSimulator` (TimedAction) drips a canned prompt every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Produces a structured image description on a prompt; accepts prior policy feedback on revisions. | `ScoringWorkflow` | returns `ImageDescription` to workflow |
| `ScorerAgent` | `AutonomousAgent` | Evaluates a description against the policy rubric; returns `PASS` or `FAIL` with notes. | `ScoringWorkflow` | returns `PolicyVerdict` to workflow |
| `ScoringWorkflow` | `Workflow` | Runs the generate → gate → score → revise loop; halts at the ceiling. | `ImageEndpoint`, `PromptConsumer` | `ImageEntity` |
| `ImageEntity` | `EventSourcedEntity` | Holds the image lifecycle, every attempt, every policy verdict, and the final outcome. | `ScoringWorkflow` | `ImagesView` |
| `PromptQueue` | `EventSourcedEntity` | Logs each submitted prompt for replay and audit. | `ImageEndpoint`, `PromptSimulator` | `PromptConsumer` |
| `ImagesView` | `View` | List-of-images read model. | `ImageEntity` events | `ImageEndpoint` |
| `PromptConsumer` | `Consumer` | Subscribes to `PromptQueue` events; starts a workflow per submission. | `PromptQueue` events | `ScoringWorkflow` |
| `PromptSimulator` | `TimedAction` | Drips a sample prompt every 60 s from `sample-events/image-prompts.jsonl`. | scheduler | `PromptQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `ImagesView`, records a `PolicyEvalRecorded` event for any cycle that completed since the last tick. | scheduler | `ImageEntity` |
| `ImageEndpoint` | `HttpEndpoint` | `/api/images/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `ImagesView`, `PromptQueue`, `ImageEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ImagePrompt(String promptText, String audienceTier, String requestedBy) {}

record ImageDescription(
    String descriptionText,
    String contentCategory,
    String brandSafetySignal,
    Instant generatedAt
) {}

record SafetyGateVerdict(boolean passed, String reasonCode, String detail) {}

record PolicyNotes(List<String> bullets, String overallRationale) {}

record PolicyVerdict(
    ScorerDecision decision,
    PolicyNotes notes,
    int score,
    Instant scoredAt
) {}

record Attempt(
    int attemptNumber,
    ImageDescription description,
    SafetyGateVerdict gateVerdict,
    Optional<PolicyVerdict> policyVerdict
) {}

record Image(
    String imageId,
    String promptText,
    String audienceTier,
    int maxAttempts,
    ImageStatus status,
    List<Attempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<String> approvedDescription,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ImageStatus { GENERATING, SCORING, APPROVED, REJECTED_FINAL }

enum ScorerDecision { PASS, FAIL }
```

### Events (on `ImageEntity`)

`ImageCreated`, `AttemptGenerated`, `AttemptGateVerdictRecorded`, `AttemptScored`, `ImageApproved`, `ImageRejectedFinal`, `PolicyEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/images` — body `{ promptText, audienceTier?, requestedBy? }` → `{ imageId }`. Starts a workflow.
- `GET /api/images` — list all images. Optional `?status=GENERATING|SCORING|APPROVED|REJECTED_FINAL`.
- `GET /api/images/{id}` — one image (including every attempt and every policy verdict).
- `GET /api/images/sse` — server-sent events stream of every image change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Image Policy Scorer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a prompt, live list of images with status pills, click-to-expand per-attempt timeline showing each description, the safety gate verdict, the scorer's verdict, and the policy notes.

Browser title: `<title>Akka Sample: Image Policy Scorer</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — safety gate guardrail** (`after-llm-response` on `GeneratorAgent`): a deterministic check that the description's `brandSafetySignal` does not appear on the prohibited-signal list. Descriptions that match are short-circuited back to the Generator with a structured feedback note (`reasonCode = PROHIBITED_SIGNAL`); they never reach the Scorer. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's scorer decision is recorded as a `PolicyEvalRecorded` event with `{ attemptNumber, decision, score, gateBlocked }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/images/{id}`.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Produces a structured image description on the prompt; on a revision call, takes the prior `PolicyNotes` as input and produces a revised description.
- `ScorerAgent` → `prompts/scorer.md`. Evaluates a description against the fixed policy rubric; returns `PASS` with a one-line rationale or `FAIL` with up to three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a prompt; image progresses `GENERATING` → `SCORING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's description and policy verdict.
2. **J2 — halt at ceiling** — Submit a prompt whose rubric is impossible to satisfy (test mode forces the Scorer to `FAIL` every attempt); image progresses through every attempt and lands in `REJECTED_FINAL` with the best description preserved and a structured rejection reason.
3. **J3 — safety gate block** — Submit a prompt that elicits a prohibited brand-safety signal; the gate short-circuits with `reasonCode = PROHIBITED_SIGNAL`, the Generator re-describes without the signal, and the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any image shows one `PolicyEvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named image-policy-scorer demonstrating the evaluator-optimizer ×
content-editorial cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-content-editorial-image-policy-scorer.
Java package io.akka.samples.imagescoring. Akka 3.6.0. HTTP port 9987.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_DESCRIPTION).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns ImageDescription{descriptionText,
    contentCategory, brandSafetySignal, generatedAt} for both GENERATE and
    REVISE_DESCRIPTION. The REVISE_DESCRIPTION task takes (originalPrompt, priorDescription,
    PolicyNotes) as inputs.
  * ScorerAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_POLICY).maxIterationsPerTask(2)).
    System prompt from prompts/scorer.md. Returns PolicyVerdict{decision, notes, score,
    scoredAt} where decision is the ScorerDecision enum (PASS | FAIL) and score is a
    1–5 integer rubric.

- 1 Workflow ScoringWorkflow with steps:
    startStep -> generateStep -> gateStep ->
    [gate FAIL? generateStep again with structured feedback : scoreStep] ->
    [decision PASS? approveStep : (attemptCount < maxAttempts ?
       generateStep with policy notes attached : rejectStep)] -> END.
  generateStep calls forAutonomousAgent(GeneratorAgent.class, imageId).runSingleTask(
    GENERATE or REVISE_DESCRIPTION) then forTask(taskId).result(GENERATE or
    REVISE_DESCRIPTION). scoreStep calls forAutonomousAgent(ScorerAgent.class,
    imageId).runSingleTask(EVALUATE_POLICY). approveStep emits ImageApproved.
    rejectStep emits ImageRejectedFinal with the highest-scoring attempt's
    description as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on generateStep and scoreStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  gateStep is a pure-function step (no LLM call): checks that
    description.brandSafetySignal() is not in the prohibited-signal set (loaded from
    application.conf as a string list). On FAIL, emits
    AttemptGateVerdictRecorded with verdict.passed = false and
    reasonCode = "PROHIBITED_SIGNAL", then transitions back to generateStep with a
    structured feedback PolicyNotes("Description contains a prohibited brand-safety
    signal; revise and resubmit.").

- 1 EventSourcedEntity ImageEntity holding state Image{imageId, promptText,
  audienceTier, maxAttempts, ImageStatus status, List<Attempt> attempts,
  Optional<Integer> approvedAttemptNumber, Optional<String> approvedDescription,
  Optional<String> rejectionReason, Instant createdAt, Optional<Instant>
  finishedAt}. ImageStatus enum: GENERATING, SCORING, APPROVED,
  REJECTED_FINAL. Events: ImageCreated, AttemptGenerated,
  AttemptGateVerdictRecorded, AttemptScored, ImageApproved,
  ImageRejectedFinal, PolicyEvalRecorded. Commands: createImage, recordGeneration,
  recordGateVerdict, recordScore, approve, rejectFinal, recordEval,
  getImage. emptyState() returns Image.initial("", "", "general", 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity PromptQueue with command enqueuePrompt(promptText,
  audienceTier, requestedBy) emitting PromptSubmitted{imageId, promptText,
  audienceTier, requestedBy, submittedAt}.

- 1 View ImagesView with row type ImageRow (mirrors Image; the attempts list
  is preserved as-is — the list is bounded at maxAttempts so size stays
  reasonable). Table updater consumes ImageEntity events. ONE query
  getAllImages SELECT * AS images FROM images_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer PromptConsumer subscribed to PromptQueue events; on
  PromptSubmitted starts a ScoringWorkflow with the imageId as the
  workflow id.

- 2 TimedActions:
  * PromptSimulator — every 60s, reads next line from
    src/main/resources/sample-events/image-prompts.jsonl and calls
    PromptQueue.enqueuePrompt.
  * EvalSampler — every 30s, queries ImagesView.getAllImages, finds images
    with a scored attempt that has not yet been recorded as a
    PolicyEvalRecorded event, and calls ImageEntity.recordEval(attemptNumber,
    decision, score, gateBlocked). Idempotent per (imageId, attemptNumber).

- 2 HttpEndpoints:
  * ImageEndpoint at /api with POST /images, GET /images, GET /images/{id},
    GET /images/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /images body
    is {promptText, audienceTier?, requestedBy?}; missing audienceTier
    defaults to "general", missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ImageTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  ImageDescription), REVISE_DESCRIPTION (ImageDescription), EVALUATE_POLICY (PolicyVerdict).
- Domain records ImageDescription, SafetyGateVerdict, PolicyNotes, PolicyVerdict,
  Attempt, Image; enums ImageStatus, ScorerDecision.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9987 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  image-policy-scorer.scoring.max-attempts = 4 and
  image-policy-scorer.scoring.prohibited-signals = ["violence", "adult-explicit",
  "hate-speech", "self-harm"], overridable by env var.
- src/main/resources/sample-events/image-prompts.jsonl with 8 canned prompt
  lines, each shaped {"promptText":"...", "audienceTier":"general"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 safety-gate guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching simplified_view
  list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = image-compliance-iteration,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.content-generation = true, capabilities.synthetic-media = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/scorer.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Image Policy Scorer",
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
  <title>Akka Sample: Image Policy Scorer</title>.

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
  named in Section 9: generator.json, scorer.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 ImageDescription entries. Three are "first-pass"
      descriptions with safe brandSafetySignal values ("none", "mild")
      and content categories ("product", "lifestyle", "nature"). Two are
      "revision" descriptions that reference the prior policy notes
      (adjusted category, neutralised signal). One is an intentionally
      prohibited description (brandSafetySignal = "violence") used to
      exercise the safety gate in J3.
    scorer.json — 6 PolicyVerdict entries. Three return decision=PASS with
      score=4 or 5 and a one-sentence rationale. Three return
      decision=FAIL with score=2 or 3 and a PolicyNotes payload of three
      bullets ("brand-safety signal is ambiguous", "audience tier mismatch
      for content category", "platform policy violation on graphic detail").
- A MockModelProvider.seedFor(imageId, attemptNumber) helper makes the
  selection deterministic per (imageId, attemptNumber) so the same image in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeneratorAgent
  and ScorerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an ImageTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Image row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ImageTasks.java is mandatory; generating GeneratorAgent or
  ScorerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9987, declared in application.conf
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
