# SPEC — on-brand-genmedia

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** On-Brand GenMedia.
**One-line pitch:** Submit a campaign brief; a brand agent generates ad copy and social captions; a reviewer agent scores the output against a configurable brand rubric; the two iterate until the reviewer approves the asset or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`BrandAgent`) and a reviewer agent (`ReviewerAgent`), feeding each review back into the next generation attempt until approval or a halt. The blueprint also demonstrates three governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, a **guardrail** that inspects each generated asset after the LLM responds and before the reviewer runs (blocking prohibited words and off-brand claims), and a **halt** that ends the loop gracefully at the retry ceiling without leaving the asset in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a campaign brief (a product name, a channel, a tone, and an optional token ceiling).

1. The system creates an `Asset` record in `GENERATING` and starts a `GenerationWorkflow`.
2. The brand agent drafts attempt #1: ad headline, body copy, and a social caption for the specified channel.
3. The after-response guardrail inspects the generated text for prohibited language (competitor names, unsubstantiated superlatives, regulatory trigger words). Failing assets are short-circuited back to the brand agent with a deterministic feedback note; they never reach the reviewer.
4. The reviewer scores the asset against a fixed rubric (brand voice, claim accuracy, channel fit, compliance clarity) and returns either `APPROVE` with a one-line rationale, or `REVISE` with a typed `ReviewNotes` payload (three bullets at most).
5. On `APPROVE`, the workflow transitions the asset to `APPROVED` with the winning attempt's text and the reviewer's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the review, and the reviewer's verdict on the entity, then calls the brand agent again with the review attached. The brand agent produces attempt #2.
7. If the loop reaches `maxAttempts` (default 4) without an `APPROVE`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the highest-scoring attempt is preserved on the entity along with every review for audit, and a `ReviewEvalRecorded` event captures the rejection.

A `CampaignSimulator` (TimedAction) drips a canned brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BrandAgent` | `AutonomousAgent` | Generates ad copy and social captions on a campaign brief; incorporates prior reviewer notes on revision calls. | `GenerationWorkflow` | returns `GeneratedAsset` to workflow |
| `ReviewerAgent` | `AutonomousAgent` | Scores an asset against the brand rubric; returns `APPROVE` or `REVISE` with notes. | `GenerationWorkflow` | returns `Review` to workflow |
| `GenerationWorkflow` | `Workflow` | Runs the generate → guardrail → review → revise loop; halts at the ceiling. | `BrandEndpoint`, `CampaignRequestConsumer` | `AssetEntity` |
| `AssetEntity` | `EventSourcedEntity` | Holds the asset lifecycle, every generation attempt, every review, and the final outcome. | `GenerationWorkflow` | `AssetsView` |
| `CampaignQueue` | `EventSourcedEntity` | Logs each submitted campaign brief for replay and audit. | `BrandEndpoint`, `CampaignSimulator` | `CampaignRequestConsumer` |
| `AssetsView` | `View` | List-of-assets read model. | `AssetEntity` events | `BrandEndpoint` |
| `CampaignRequestConsumer` | `Consumer` | Subscribes to `CampaignQueue` events; starts a workflow per submission. | `CampaignQueue` events | `GenerationWorkflow` |
| `CampaignSimulator` | `TimedAction` | Drips a sample brief every 60 s from `sample-events/campaign-briefs.jsonl`. | scheduler | `CampaignQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `AssetsView`, records a `ReviewEvalRecorded` event for any cycle that completed since the last tick. | scheduler | `AssetEntity` |
| `BrandEndpoint` | `HttpEndpoint` | `/api/assets/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `AssetsView`, `CampaignQueue`, `AssetEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record CampaignBrief(
    String product,
    String channel,
    String tone,
    int tokenCeiling,
    String requestedBy
) {}

record GeneratedAsset(
    String headline,
    String bodyCopy,
    String socialCaption,
    int tokenCount,
    Instant generatedAt
) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record ReviewNotes(List<String> bullets, String overallRationale) {}

record Review(
    ReviewerVerdict verdict,
    ReviewNotes notes,
    int score,
    Instant reviewedAt
) {}

record Attempt(
    int attemptNumber,
    GeneratedAsset asset,
    GuardrailVerdict guardrail,
    Optional<Review> review
) {}

record Asset(
    String assetId,
    String product,
    String channel,
    String tone,
    int tokenCeiling,
    int maxAttempts,
    AssetStatus status,
    List<Attempt> attempts,
    Optional<Integer> approvedAttemptNumber,
    Optional<GeneratedAsset> approvedAsset,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AssetStatus { GENERATING, REVIEWING, APPROVED, REJECTED_FINAL }

enum ReviewerVerdict { APPROVE, REVISE }
```

### Events (on `AssetEntity`)

`AssetCreated`, `AttemptGenerated`, `AttemptGuardrailVerdictRecorded`, `AttemptReviewed`, `AssetApproved`, `AssetRejectedFinal`, `ReviewEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/assets` — body `{ product, channel, tone?, tokenCeiling?, requestedBy? }` → `{ assetId }`. Starts a workflow.
- `GET /api/assets` — list all assets. Optional `?status=GENERATING|REVIEWING|APPROVED|REJECTED_FINAL`.
- `GET /api/assets/{id}` — one asset (including every attempt and every review).
- `GET /api/assets/sse` — server-sent events stream of every asset change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "On-Brand GenMedia"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red, halt = red).
- **App UI** — form to submit a campaign brief, live list of assets with status pills, click-to-expand per-attempt timeline showing each generated asset, the guardrail verdict, the reviewer's verdict, and the reviewer's notes.

Browser title: `<title>Akka Sample: On-Brand GenMedia</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — brand/safety guardrail** (`after-llm-response` on `BrandAgent`): a deterministic check that the generated text contains no prohibited words (competitor names, unsubstantiated superlatives such as "world's best", or regulatory trigger phrases such as "guaranteed results"). Failing assets are short-circuited back to the brand agent with a structured feedback note (`reasonCode = PROHIBITED_CONTENT`); they never reach the reviewer. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's review is recorded as a `ReviewEvalRecorded` event with `{ attemptNumber, verdict, score, guardrailFailed }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits one event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/assets/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without an `APPROVE`, the workflow ends with `AssetRejectedFinal`. The entity preserves every attempt, every review, the highest-scoring attempt's asset, and a structured rejection reason. The system never deletes generated content or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `BrandAgent` → `prompts/brand-agent.md`. Generates a media asset (headline + body copy + social caption) for the campaign brief; on a revision call, takes the prior `ReviewNotes` as input and produces a new asset.
- `ReviewerAgent` → `prompts/reviewer-agent.md`. Scores an asset against the fixed brand rubric; returns `APPROVE` with a one-line rationale or `REVISE` with three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a campaign brief; asset progresses `GENERATING` → `REVIEWING` → `APPROVED` within the retry ceiling; the App UI shows every attempt's generated content and review.
2. **J2 — halt at ceiling** — Submit a brief whose rubric is impossible (test mode forces the reviewer to `REVISE` every attempt); asset progresses through every attempt and lands in `REJECTED_FINAL` with the best asset preserved and a structured rejection reason.
3. **J3 — guardrail block** — Submit a brief that causes the brand agent to include a prohibited term; the guardrail short-circuits with `reasonCode = PROHIBITED_CONTENT`, the brand agent re-generates a clean version, the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any asset shows one `ReviewEvalRecorded` event per reviewed attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named on-brand-genmedia demonstrating the evaluator-optimizer ×
sales-marketing cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-sales-marketing-on-brand-media.
Java package io.akka.samples.onbrandgenmedia. Akka 3.6.0. HTTP port 9536.

Components to wire (exactly):
- 2 AutonomousAgents:
  * BrandAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ASSET).maxIterationsPerTask(3)).
    System prompt loaded from prompts/brand-agent.md. Returns GeneratedAsset{headline,
    bodyCopy, socialCaption, tokenCount, generatedAt} for both GENERATE and
    REVISE_ASSET. The REVISE_ASSET task takes (originalBrief, priorAsset,
    ReviewNotes) as inputs.
  * ReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/reviewer-agent.md. Returns Review{verdict, notes, score,
    reviewedAt} where verdict is the ReviewerVerdict enum (APPROVE | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow GenerationWorkflow with steps:
    startStep -> generateStep -> guardrailStep -> [guardrail FAIL? generateStep
    again with structured feedback : reviewStep] ->
    [verdict APPROVE? approveStep : (attemptCount < maxAttempts ?
       generateStep with review attached : rejectStep)] -> END.
  generateStep calls forAutonomousAgent(BrandAgent.class, assetId).runSingleTask(
    GENERATE or REVISE_ASSET) then forTask(taskId).result(GENERATE or
    REVISE_ASSET). reviewStep calls forAutonomousAgent(ReviewerAgent.class,
    assetId).runSingleTask(REVIEW). approveStep emits AssetApproved.
    rejectStep emits AssetRejectedFinal with the highest-scoring attempt's
    asset as best-of and a structured rejectionReason. Override settings()
    with stepTimeout(60s) on generateStep and reviewStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): scans the generated
    text for prohibited content. On FAIL, emits
    AttemptGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "PROHIBITED_CONTENT", then transitions back to generateStep
    with a structured feedback ReviewNotes("Generated content contains
    prohibited terms; revise to remove them.").

- 1 EventSourcedEntity AssetEntity holding state Asset{assetId, product,
  channel, tone, tokenCeiling, maxAttempts, AssetStatus status,
  List<Attempt> attempts, Optional<Integer> approvedAttemptNumber,
  Optional<GeneratedAsset> approvedAsset, Optional<String> rejectionReason,
  Instant createdAt, Optional<Instant> finishedAt}. AssetStatus enum:
  GENERATING, REVIEWING, APPROVED, REJECTED_FINAL. Events: AssetCreated,
  AttemptGenerated, AttemptGuardrailVerdictRecorded, AttemptReviewed,
  AssetApproved, AssetRejectedFinal, ReviewEvalRecorded. Commands:
  createAsset, recordGeneration, recordGuardrail, recordReview, approve,
  rejectFinal, recordEval, getAsset. emptyState() returns Asset.initial("")
  with no commandContext() reference. Event-applier wraps lifecycle fields
  with Optional.of(...).

- 1 EventSourcedEntity CampaignQueue with command enqueueBrief(product,
  channel, tone, tokenCeiling, requestedBy) emitting
  BriefSubmitted{assetId, product, channel, tone, tokenCeiling, requestedBy,
  submittedAt}.

- 1 View AssetsView with row type AssetRow (mirrors Asset; the attempts list
  is preserved as-is — the list is bounded at maxAttempts so size stays
  reasonable). Table updater consumes AssetEntity events. ONE query
  getAllAssets SELECT * AS assets FROM assets_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer CampaignRequestConsumer subscribed to CampaignQueue events; on
  BriefSubmitted starts a GenerationWorkflow with the assetId as the
  workflow id.

- 2 TimedActions:
  * CampaignSimulator — every 60s, reads next line from
    src/main/resources/sample-events/campaign-briefs.jsonl and calls
    CampaignQueue.enqueueBrief.
  * EvalSampler — every 30s, queries AssetsView.getAllAssets, finds assets
    with a reviewed attempt that has not yet been recorded as a
    ReviewEvalRecorded event, and calls AssetEntity.recordEval(attemptNumber,
    verdict, score, guardrailFailed). Idempotent per (assetId, attemptNumber).

- 2 HttpEndpoints:
  * BrandEndpoint at /api with POST /assets, GET /assets, GET /assets/{id},
    GET /assets/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /assets body
    is {product, channel, tone?, tokenCeiling?, requestedBy?}; missing
    tokenCeiling defaults to 300, missing tone defaults to "professional",
    missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- BrandTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  GeneratedAsset), REVISE_ASSET (GeneratedAsset), REVIEW (Review).
- Domain records GeneratedAsset, GuardrailVerdict, ReviewNotes, Review,
  Attempt, Asset; enums AssetStatus, ReviewerVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9536 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  on-brand-genmedia.generation.max-attempts = 4 and
  on-brand-genmedia.generation.default-token-ceiling = 300, overridable by
  env var.
- src/main/resources/sample-events/campaign-briefs.jsonl with 8 canned brief
  lines, each shaped {"product":"...","channel":"...","tone":"professional","tokenCeiling":300}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 brand/safety guardrail
  after-llm-response, E1 eval-event on-decision-eval, HT1 halt
  graceful-degradation) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = brand-content-generation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.synthetic-media = true, capabilities.content-generation = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/brand-agent.md, prompts/reviewer-agent.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: On-Brand GenMedia",
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
  <title>Akka Sample: On-Brand GenMedia</title>.

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
  named in Section 9: brand-agent.json, reviewer-agent.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    brand-agent.json — 6 GeneratedAsset entries. Three are first-pass
      assets with headlines under 12 words, professional body copy under
      300 tokens, and a social caption under 40 words covering the products
      in campaign-briefs.jsonl. Two are revision assets that address a prior
      review note (tighter headline, removed superlative). One intentionally
      contains a prohibited term ("world's best") to exercise the guardrail
      in J3.
    reviewer-agent.json — 6 Review entries. Three return verdict=APPROVE
      with score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a ReviewNotes payload of three
      bullets ("Headline overpromises", "Body copy missing call-to-action",
      "Social caption exceeds 40 words").
- A MockModelProvider.seedFor(assetId, attemptNumber) helper makes the
  selection deterministic per (assetId, attemptNumber) so the same asset in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. BrandAgent
  and ReviewerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a BrandTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Asset row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: BrandTasks.java is mandatory; generating BrandAgent or
  ReviewerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9536, declared in application.conf
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
