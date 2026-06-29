# SPEC — mr-reviewer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** GitLab MR Reviewer.
**One-line pitch:** Push a merge request webhook; a reviewer agent produces structured findings; a gate agent checks thoroughness; the two iterate until the gate passes or the loop hits its retry ceiling — then a CI signal is posted back.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`ReviewerAgent`) and a reviewer agent (`GateAgent`), feeding each gate verdict back into the next review pass until convergence or a halt. The blueprint also demonstrates three governance mechanisms — a **secret sanitizer** that strips credential patterns from diffs before any LLM call, an **output guardrail** that blocks review comments reproducing secrets or verbatim proprietary code before the comment is posted to GitLab, and a **CI gate** that publishes a machine-readable pass/fail signal that CI pipelines can consume.

## 3. User-facing flows

The user opens the App UI tab and submits a simulated MR webhook (project path, MR IID, and a raw diff string).

1. The system creates an `MergeRequest` record in `RECEIVED` and starts a `ReviewWorkflow`.
2. The secret sanitizer scans the diff for credential patterns (API keys, private-key headers, high-entropy tokens). If a pattern is found, the MR transitions to `SANITIZER_BLOCKED` and no LLM call is made. Otherwise the diff advances to the reviewer.
3. `ReviewerAgent` produces `ReviewPass #1`: a `ReviewResult` containing per-file findings, a summary paragraph, and a numeric quality score (0–100).
4. The output guardrail checks `ReviewResult.commentary` for patterns that reproduce secrets or verbatim proprietary code. Blocked commentary is short-circuited back to the reviewer with a deterministic feedback note; it never reaches the gate.
5. `GateAgent` scores the review against a fixed rubric (coverage of changed files, actionability of findings, non-reproduction of secrets, overall thoroughness) and returns either `PASS` with a one-line rationale, or `REFINE` with a typed `GateFeedback` payload (up to three bullets).
6. On `PASS`, the workflow transitions the MR to `CI_PASS`, records the accepted review, and publishes a CI signal.
7. On `REFINE`, the workflow records the pass, the gate feedback, and the guardrail verdict on the entity, then calls `ReviewerAgent` again with the gate feedback attached. The reviewer produces pass #2.
8. If the loop reaches `maxPasses` (default 3) without a `PASS`, the halt mechanism activates: the workflow ends with `CI_FAIL`, the highest-scoring pass is preserved on the entity along with every gate verdict for audit, and a `ReviewEvalRecorded` event captures the rejection.

An `MrSimulator` (TimedAction) drips a canned MR webhook every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReviewerAgent` | `AutonomousAgent` | Reads a diff and produces structured review commentary; accepts prior gate feedback on refinement passes. | `ReviewWorkflow` | returns `ReviewResult` to workflow |
| `GateAgent` | `AutonomousAgent` | Scores a review against the rubric; returns `PASS` or `REFINE` with feedback. | `ReviewWorkflow` | returns `GateVerdict` to workflow |
| `ReviewWorkflow` | `Workflow` | Runs the diff → sanitize → review → guardrail → gate → refine loop; halts at the ceiling; publishes CI signal. | `ReviewEndpoint`, `WebhookConsumer` | `MrEntity` |
| `MrEntity` | `EventSourcedEntity` | Holds the MR lifecycle, every review pass, every gate verdict, and the final CI signal. | `ReviewWorkflow` | `MrView` |
| `MrQueue` | `EventSourcedEntity` | Logs each incoming webhook for replay and audit. | `ReviewEndpoint`, `MrSimulator` | `WebhookConsumer` |
| `MrView` | `View` | List-of-MRs read model. | `MrEntity` events | `ReviewEndpoint` |
| `WebhookConsumer` | `Consumer` | Subscribes to `MrQueue` events; starts a workflow per MR submission. | `MrQueue` events | `ReviewWorkflow` |
| `MrSimulator` | `TimedAction` | Drips a sample MR webhook every 90 s from `sample-events/mr-webhooks.jsonl`. | scheduler | `MrQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `MrView`, records a `ReviewEvalRecorded` event for any pass that completed since the last tick. | scheduler | `MrEntity` |
| `ReviewEndpoint` | `HttpEndpoint` | `/api/mrs/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `MrView`, `MrQueue`, `MrEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record MrWebhook(String projectPath, int mrIid, String targetBranch, String diffText, String submittedBy) {}

record SanitizerVerdict(boolean passed, String reasonCode, String detail) {}

record Finding(String filePath, int lineNumber, String severity, String message) {}

record ReviewResult(
    List<Finding> findings,
    String summary,
    int qualityScore,
    String commentary,
    Instant reviewedAt
) {}

record GateFeedback(List<String> bullets, String overallRationale) {}

record GateVerdict(GateDecision decision, GateFeedback feedback, int gateScore, Instant evaluatedAt) {}

record ReviewPass(
    int passNumber,
    SanitizerVerdict sanitizer,
    Optional<ReviewResult> review,
    Optional<GateVerdict> gate
) {}

record MergeRequest(
    String mrId,
    String projectPath,
    int mrIid,
    String targetBranch,
    int maxPasses,
    MrStatus status,
    List<ReviewPass> passes,
    Optional<Integer> acceptedPassNumber,
    Optional<ReviewResult> acceptedReview,
    Optional<String> ciSignal,
    Instant receivedAt,
    Optional<Instant> finishedAt
) {}

enum MrStatus { RECEIVED, REVIEWING, GATE_CHECKING, CI_PASS, CI_FAIL, SANITIZER_BLOCKED }

enum GateDecision { PASS, REFINE }
```

### Events (on `MrEntity`)

`MrReceived`, `SanitizerVerdictRecorded`, `ReviewPassDrafted`, `ReviewGuardrailVerdictRecorded`, `ReviewPassGated`, `MrCiPassed`, `MrCiFailed`, `ReviewEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/mrs` — body `{ projectPath, mrIid, targetBranch, diffText, submittedBy? }` → `{ mrId }`. Starts a workflow.
- `GET /api/mrs` — list all MRs. Optional `?status=RECEIVED|REVIEWING|GATE_CHECKING|CI_PASS|CI_FAIL|SANITIZER_BLOCKED`.
- `GET /api/mrs/{id}` — one MR (including every review pass and every gate verdict).
- `GET /api/mrs/sse` — server-sent events stream of every MR change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "GitLab MR Reviewer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (ci-gate = blue, guardrail = red, sanitizer = red).
- **App UI** — form to submit an MR webhook, live list of MRs with status pills, click-to-expand per-pass timeline showing each review, the sanitizer verdict, the guardrail verdict, the gate verdict, and the gate feedback.

Browser title: `<title>Akka Sample: GitLab MR Reviewer</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — secret sanitizer** (`system-level`): a deterministic scan of the raw diff for credential patterns (BEGIN PRIVATE KEY, api_key=, high-entropy hex/base64 tokens ≥ 32 chars, AWS key prefixes). A matching diff records `SanitizerVerdict.passed = false` with `reasonCode = SECRET_DETECTED`, transitions the MR to `SANITIZER_BLOCKED`, and ends the workflow without any LLM call. Enforcement: system-level.
- **G1 — output guardrail** (`before-agent-response` on `ReviewerAgent`): a deterministic check that `ReviewResult.commentary` does not contain patterns matching the sanitizer's ruleset. Blocked commentary is short-circuited back to the reviewer with a structured feedback note (`reasonCode = COMMENTARY_REPRODUCES_SECRET`); it never reaches `GateAgent`. Enforcement: blocking.
- **CG1 — CI gate** (`build-gate`): when the workflow reaches a terminal state, it publishes a typed `CiSignal` (`CI_PASS` or `CI_FAIL`) on `MrEntity` which the `ReviewEndpoint` surfaces at `GET /api/mrs/{id}/ci-signal`. External CI pipelines poll or SSE-subscribe to this endpoint to drive pipeline gating. Enforcement: build-gate.

## 9. Agent prompts

- `ReviewerAgent` → `prompts/reviewer.md`. Reviews a diff and produces structured findings; on a refinement call, takes the prior `GateFeedback` as input and produces an updated `ReviewResult`.
- `GateAgent` → `prompts/gate.md`. Scores a review against the fixed rubric; returns `PASS` with a one-line rationale or `REFINE` with up to three short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a diff webhook; MR progresses `RECEIVED` → `REVIEWING` → `GATE_CHECKING` → `CI_PASS` within the retry ceiling; the App UI shows every pass's review and gate verdict.
2. **J2 — halt at ceiling** — Submit a diff whose rubric is impossible (test mode forces the Gate to `REFINE` every pass); MR progresses through every pass and lands in `CI_FAIL` with the best review preserved and a structured CI signal.
3. **J3 — sanitizer block** — Submit a diff containing a credential pattern; the sanitizer records `SECRET_DETECTED`, the MR transitions to `SANITIZER_BLOCKED`, no LLM call is made, and the App UI shows the red `SECRET_DETECTED` pill.
4. **J4 — output guardrail block** — Submit a diff where the reviewer's mock response reproduces a secret; the guardrail records `COMMENTARY_REPRODUCES_SECRET`, the reviewer is retried with structured feedback, and the critic never sees the leaking commentary.
5. **J5 — eval-event timeline** — The expanded view of any MR shows one `ReviewEvalRecorded` event per completed pass and one terminal event on CI resolution.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mr-reviewer demonstrating the evaluator-optimizer ×
dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-dev-code-mr-reviewer.
Java package io.akka.samples.gitlabcodereviewwithchatgpt. Akka 3.6.0.
HTTP port 9629.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ReviewerAgent — definition() with
    capability(TaskAcceptance.of(REVIEW_DIFF).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REFINE_REVIEW).maxIterationsPerTask(3)).
    System prompt loaded from prompts/reviewer.md. Returns ReviewResult{findings,
    summary, qualityScore, commentary, reviewedAt} for both REVIEW_DIFF and
    REFINE_REVIEW. The REFINE_REVIEW task takes (originalDiff, priorReview,
    GateFeedback) as inputs.
  * GateAgent — definition() with
    capability(TaskAcceptance.of(GATE_REVIEW).maxIterationsPerTask(2)). System
    prompt from prompts/gate.md. Returns GateVerdict{decision, feedback, gateScore,
    evaluatedAt} where decision is the GateDecision enum (PASS | REFINE)
    and gateScore is a 0–100 integer.

- 1 Workflow ReviewWorkflow with steps:
    startStep -> sanitizerStep -> [sanitizer FAIL? blockStep : reviewStep] ->
    guardrailStep -> [guardrail FAIL? reviewStep again with structured feedback :
    gateStep] -> [decision PASS? ciPassStep : (passCount < maxPasses ?
       reviewStep with gate feedback : ciFailStep)] -> END.
  reviewStep calls forAutonomousAgent(ReviewerAgent.class, mrId).runSingleTask(
    REVIEW_DIFF or REFINE_REVIEW) then forTask(taskId).result(REVIEW_DIFF or
    REFINE_REVIEW). gateStep calls forAutonomousAgent(GateAgent.class,
    mrId).runSingleTask(GATE_REVIEW). ciPassStep emits MrCiPassed.
    ciFailStep emits MrCiFailed with the highest-scoring pass as best-of.
    blockStep emits SanitizerVerdictRecorded with passed=false and ends workflow.
    Override settings() with stepTimeout(60s) on reviewStep and gateStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(ciFailStep)).
  sanitizerStep is a pure-function step: scans diffText for patterns
    (BEGIN PRIVATE KEY, api_key=, AWS key prefixes, high-entropy tokens ≥ 32 chars).
    On FAIL, emits SanitizerVerdictRecorded with reasonCode = "SECRET_DETECTED"
    and transitions to blockStep.
  guardrailStep is a pure-function step: checks that ReviewResult.commentary
    does not reproduce any of the sanitizer patterns. On FAIL, emits
    ReviewGuardrailVerdictRecorded with reasonCode = "COMMENTARY_REPRODUCES_SECRET",
    then transitions back to reviewStep with a structured GateFeedback
    ("Review commentary reproduces a detected secret; rewrite without quoting
    sensitive tokens.").

- 1 EventSourcedEntity MrEntity holding state MergeRequest{mrId, projectPath,
  mrIid, targetBranch, maxPasses, MrStatus status, List<ReviewPass> passes,
  Optional<Integer> acceptedPassNumber, Optional<ReviewResult> acceptedReview,
  Optional<String> ciSignal, Instant receivedAt, Optional<Instant> finishedAt}.
  MrStatus enum: RECEIVED, REVIEWING, GATE_CHECKING, CI_PASS, CI_FAIL,
  SANITIZER_BLOCKED. Events: MrReceived, SanitizerVerdictRecorded,
  ReviewPassDrafted, ReviewGuardrailVerdictRecorded, ReviewPassGated,
  MrCiPassed, MrCiFailed, ReviewEvalRecorded. Commands: receiveMr,
  recordSanitizer, recordReviewPass, recordGuardrail, recordGateVerdict,
  ciPass, ciFail, recordEval, getMr. emptyState() returns
  MergeRequest.initial("", "", 0, "", 3) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity MrQueue with command enqueueWebhook(projectPath,
  mrIid, targetBranch, diffText, submittedBy) emitting
  WebhookReceived{mrId, projectPath, mrIid, targetBranch, diffText,
  submittedBy, receivedAt}.

- 1 View MrView with row type MrRow (mirrors MergeRequest; the passes list
  is preserved as-is — the list is bounded at maxPasses so size stays
  reasonable). Table updater consumes MrEntity events. ONE query
  getAllMrs SELECT * AS mrs FROM mr_view. No WHERE status filter — caller
  filters client-side (Lesson 2).

- 1 Consumer WebhookConsumer subscribed to MrQueue events; on
  WebhookReceived starts a ReviewWorkflow with the mrId as the workflow id.

- 2 TimedActions:
  * MrSimulator — every 90s, reads next line from
    src/main/resources/sample-events/mr-webhooks.jsonl and calls
    MrQueue.enqueueWebhook.
  * EvalSampler — every 30s, queries MrView.getAllMrs, finds passes
    with a gate verdict that has not yet been recorded as a
    ReviewEvalRecorded event, and calls MrEntity.recordEval(passNumber,
    decision, gateScore, secretBlocked). Idempotent per (mrId, passNumber).

- 2 HttpEndpoints:
  * ReviewEndpoint at /api with POST /mrs, GET /mrs, GET /mrs/{id},
    GET /mrs/{id}/ci-signal, GET /mrs/sse, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /mrs body is {projectPath,
    mrIid, targetBranch, diffText, submittedBy?}; missing submittedBy
    defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ReviewTasks.java declaring three Task<R> constants: REVIEW_DIFF (resultConformsTo
  ReviewResult), REFINE_REVIEW (ReviewResult), GATE_REVIEW (GateVerdict).
- Domain records MrWebhook, SanitizerVerdict, Finding, ReviewResult, GateFeedback,
  GateVerdict, ReviewPass, MergeRequest; enums MrStatus, GateDecision.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9629 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  mr-reviewer.review.max-passes = 3 and
  mr-reviewer.review.ci-gate-threshold = 70, overridable by env var.
- src/main/resources/sample-events/mr-webhooks.jsonl with 8 canned
  webhook lines, each shaped {"projectPath":"...","mrIid":N,
  "targetBranch":"main","diffText":"..."}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (S1 sanitizer
  system-level, G1 guardrail before-agent-response, CG1 ci-gate
  build-gate) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = code-review-automation,
  decisions.authority_level = advisory-only, data.data_classes.pii = false,
  capabilities.* = false except content-generation = true; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/reviewer.md, prompts/gate.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: GitLab MR Reviewer",
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
  per-pass timeline). Browser title exactly:
  <title>Akka Sample: GitLab MR Reviewer</title>.

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
        application.conf; /akka:build forwards the value.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the Task<R> id.
  Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (reviewer.json,
  gate.json), picks one entry pseudo-randomly per call, and deserialises
  it into the agent's typed return shape.
- Per-agent mock-response shapes:
    reviewer.json — 6 ReviewResult entries. Three are solid reviews with
      2–4 findings per file, qualityScore 65–85, constructive commentary.
      Two are refinement responses that address prior gate feedback (fewer
      findings, sharper commentary, higher score). One intentionally
      contains a high-entropy token in commentary to exercise the output
      guardrail.
    gate.json — 6 GateVerdict entries. Three return decision=PASS with
      gateScore=75–90 and a one-sentence rationale. Three return
      decision=REFINE with gateScore=40–60 and a GateFeedback payload
      of three bullets ("coverage of auth.go is missing", "findings in
      main.go lack line citations", "summary does not address test changes").
- A MockModelProvider.seedFor(mrId, passNumber) helper makes the
  selection deterministic per (mrId, passNumber).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: ReviewerAgent and GateAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with a
  ReviewTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the MergeRequest row record
  is Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ReviewTasks.java is mandatory; generating ReviewerAgent or
  GateAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9629, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. Exactly five <section class="tab-panel"> elements.
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
