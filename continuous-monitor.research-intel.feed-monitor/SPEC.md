# SPEC — feed-monitor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Feed Monitor.
**One-line pitch:** A background worker polls RSS and web feeds, summarizes and classifies each item with AI, and posts selected notifications to Slack.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern with two governance mechanisms layered on top of two AI primitives (`SummaryAgent` and `ClassifierAgent`). Specifically:

- A **before-tool-call guardrail** on the `postToSlack` tool enforces that only items routed to channels in the configured allow-list can be posted. An item targeting a disallowed channel is rejected before the network call is made.
- An **eval-periodic** sampler firing every 30 minutes scores a sample of recently posted items for summary quality and classification accuracy, giving a continuous quality signal on the monitor's output.

Together these two mechanisms cover the two primary risks: unintended broadcast to the wrong Slack channel, and silent quality degradation in the AI-generated summaries over time.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live feed: every received item, its AI-generated summary, its classification, and (if notified) the Slack post record.
2. `FeedPoller` (TimedAction) ticks every 60 s and inserts new simulated feed items into `FeedItemEntity`. (A fixture-file drip — reads from `sample-events/feed-items.jsonl`.)
3. For each new item: `FeedWorkflow` runs `SummaryAgent` to produce a summary and extracted topics, then `ClassifierAgent` to decide `NOTIFY` / `SUPPRESS` / `REVIEW`.
4. If the classification is `NOTIFY`, `SlackNotifier` Consumer formats and dispatches the post. The before-tool-call guardrail validates the target channel before the Slack API is called.
5. If the classification is `SUPPRESS`, the item reaches a terminal SUPPRESSED state; no notification is sent.
6. If the classification is `REVIEW`, the item reaches a PENDING_REVIEW state and waits for a human to override via the API.
7. `EvalRunner` (TimedAction) ticks every 30 minutes, picks up to 5 POSTED items without an `evalScore`, calls `EvalJudge`, and writes a score back via an `EvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `FeedPoller` | `TimedAction` | Drips simulated feed items every 60 s. | scheduler | `FeedItemEntity` |
| `FeedItemEntity` | `EventSourcedEntity` | Full per-item lifecycle. | `FeedPoller`, `FeedEndpoint`, `SlackNotifier` | `FeedView`, `FeedWorkflow` |
| `SummaryAgent` | `Agent` (typed) | Summarizes a feed item and extracts topics. | invoked by `FeedWorkflow` | returns `SummaryResult` |
| `ClassifierAgent` | `Agent` (typed) | Scores relevance and routes to NOTIFY / SUPPRESS / REVIEW. | invoked by `FeedWorkflow` | returns `ClassificationResult` |
| `FeedWorkflow` | `Workflow` | Per-item orchestration: summarize → classify → (if NOTIFY) trigger notifier. | `FeedPoller` (one workflow per item) | `FeedItemEntity`, `SlackNotifier` |
| `SlackNotifier` | `Consumer` | Formats and dispatches outbound Slack posts; before-tool-call guardrail runs here. | `FeedItemEntity` events (NOTIFY_REQUESTED) | Slack API (stub) |
| `FeedView` | `View` | Read-model row per item. | `FeedItemEntity` events | `FeedEndpoint` |
| `EvalRunner` | `TimedAction` | Every 30 min, samples POSTED items without scores; calls judge; writes `EvalScored`. | scheduler | `FeedItemEntity` |
| `FeedEndpoint` | `HttpEndpoint` | `/api/feed/*` — list, get, override, suppress, SSE. | — | `FeedView`, `FeedItemEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record FeedItem(
    String itemId,
    String feedUrl,
    String title,
    String rawContent,
    String link,
    Instant fetchedAt
) {}

record SummaryResult(
    String summary,
    List<String> topics,
    String keyQuote,
    Instant summarizedAt
) {}

record ClassificationResult(
    FeedClassification classification,
    String confidence,
    String reason,
    Optional<String> targetChannel
) {}
enum FeedClassification { NOTIFY, SUPPRESS, REVIEW }

record SlackPost(
    String channel,
    String messageText,
    Instant postedAt,
    boolean guardrailPassed
) {}

record EvalResult(int score, String rationale) {}

record FeedItemState(
    String itemId,
    FeedItem raw,
    Optional<SummaryResult> summary,
    Optional<ClassificationResult> classification,
    Optional<SlackPost> post,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    FeedItemStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum FeedItemStatus {
    RECEIVED, SUMMARIZED, CLASSIFIED, NOTIFY_REQUESTED,
    POSTED, SUPPRESSED, PENDING_REVIEW, FAILED
}
```

Events on `FeedItemEntity`: `ItemReceived`, `ItemSummarized`, `ItemClassified`, `NotifyRequested`, `ItemPosted`, `ItemSuppressed`, `ItemPendingReview`, `ItemFailed`, `ReviewOverridden`, `EvalScored`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/feed` — list all items. Optional `?status=…`.
- `GET /api/feed/{id}` — one item.
- `POST /api/feed/{id}/override` — body `{ decidedBy, targetChannel }` → moves PENDING_REVIEW to NOTIFY_REQUESTED.
- `POST /api/feed/{id}/suppress` — body `{ decidedBy }` → moves PENDING_REVIEW to SUPPRESSED.
- `GET /api/feed/sse` — Server-Sent Events for every item change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Feed Monitor</title>`.

App UI tab is the most distinctive: it shows the **live feed** with a card per item, classified with NOTIFY/SUPPRESS/REVIEW chips, and a right-hand detail pane showing the AI summary, extracted topics, and (when PENDING_REVIEW) Override / Suppress action buttons.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on the `postToSlack` tool: checks that the target Slack channel is in the configured allow-list before the call is made. Blocking — if the channel is not allowed, the call is rejected with a structured error and the item transitions to FAILED.
- **E1 — eval-periodic** (every 30 minutes): scores POSTED items on summary quality, classification accuracy, and channel appropriateness.

## 9. Agent prompts

- `SummaryAgent` → `prompts/summary-agent.md`. Typed summarizer. Returns `SummaryResult`.
- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier. Returns `ClassificationResult` with one of the three `FeedClassification` values.
- `EvalJudge` (used by `EvalRunner`) → `prompts/eval-judge.md`. Scores a posted item on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Poller drips a feed item; it appears in the UI within 60 s; passes summarize → classify → NOTIFY_REQUESTED → POSTED.
2. **J2** — An item classified SUPPRESS never triggers a Slack post; status transitions to SUPPRESSED.
3. **J3** — An item classified REVIEW lands in PENDING_REVIEW; human clicks Override → it is posted.
4. **J4** — An item targeting a disallowed channel is blocked by the guardrail; status transitions to FAILED with a guardrail-rejection reason.
5. **J5** — Eval Runner scores at least one POSTED item within 30 minutes; the score and rationale appear in the UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named feed-monitor demonstrating the continuous-monitor × research-intel
cell. Runs out of the box (simulated feed source; no live HTTP fetch or real Slack post).
Maven group io.akka.samples, artifact continuous-monitor-research-intel-feed-monitor.
Java package io.akka.samples.aipoweredinformationmonitoringopenaisheetsjinaslack.
HTTP port 9511.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) SummaryAgent — summarizer. System prompt loaded from
  prompts/summary-agent.md. Input: FeedItem{itemId, feedUrl, title, rawContent, link,
  fetchedAt}. Output: SummaryResult{summary: String (2–4 sentences), topics: List<String>
  (up to 5 tags), keyQuote: String (best verbatim sentence from rawContent), summarizedAt}.

- 1 Agent (typed, NOT autonomous) ClassifierAgent — relevance classifier. System prompt
  from prompts/classifier-agent.md. Input: SummaryResult + FeedItem{title, feedUrl}.
  Output: ClassificationResult{classification: FeedClassification (NOTIFY/SUPPRESS/REVIEW),
  confidence: "high"|"medium"|"low", reason: String, targetChannel: Optional<String>}.
  Defaults to REVIEW under uncertainty.

- 1 AutonomousAgent EvalJudge — definition() with capability(TaskAcceptance.of(EVAL)
  .maxIterationsPerTask(2)). System prompt from prompts/eval-judge.md. Input: FeedItem +
  SummaryResult + SlackPost. Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow FeedWorkflow per item with steps: summarizeStep -> classifyStep ->
  conditional branch: NOTIFY -> requestNotifyStep -> ends; SUPPRESS -> ends with
  ItemSuppressed; REVIEW -> ends with ItemPendingReview. summarizeStep wraps SummaryAgent
  with WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(20)). classifyStep wraps
  ClassifierAgent with stepTimeout(Duration.ofSeconds(10)). requestNotifyStep emits
  NotifyRequested on FeedItemEntity; SlackNotifier Consumer picks it up asynchronously.
  On timeout in either agent step, emit ItemFailed with reason "step-timeout".

- 1 EventSourcedEntity FeedItemEntity (one per itemId). State FeedItemState{itemId,
  raw: FeedItem, Optional<SummaryResult> summary, Optional<ClassificationResult>
  classification, Optional<SlackPost> post, Optional<Integer> evalScore,
  Optional<String> evalRationale, FeedItemStatus status, Instant createdAt,
  Optional<Instant> finishedAt}. FeedItemStatus enum: RECEIVED, SUMMARIZED, CLASSIFIED,
  NOTIFY_REQUESTED, POSTED, SUPPRESSED, PENDING_REVIEW, FAILED. Events: ItemReceived,
  ItemSummarized, ItemClassified, NotifyRequested, ItemPosted, ItemSuppressed,
  ItemPendingReview, ItemFailed, ReviewOverridden, EvalScored. Commands: registerItem,
  attachSummary, attachClassification, requestNotify, markPosted, markSuppressed,
  markPendingReview, markFailed, recordReviewOverride, recordSuppressOverride,
  recordEval, getItem. emptyState() returns FeedItemState.initial("") without
  commandContext() reference.

- 1 Consumer SlackNotifier subscribed to FeedItemEntity events. For each
  NotifyRequested event: (1) validates the targetChannel against the configured
  allow-list in a before-tool-call guardrail — if the channel is not in the list,
  calls FeedItemEntity.markFailed(itemId, "guardrail:channel-not-allowed") and returns;
  (2) formats the Slack message from SummaryResult.summary + keyQuote; (3) calls the
  stub postToSlack(channel, text) no-op, records SlackPost with guardrailPassed=true,
  calls FeedItemEntity.markPosted(itemId, slackPost).

- 1 View FeedView with row type FeedItemRow (mirrors FeedItemState minus rawContent).
  Table updater consumes FeedItemEntity events. ONE query getAllItems SELECT * AS items
  FROM feed_item_view.

- 2 TimedActions:
  * FeedPoller — every 60 s, reads next line from src/main/resources/sample-events/
    feed-items.jsonl and calls FeedItemEntity.registerItem.
  * EvalRunner — every 30 minutes, queries FeedView.getAllItems, picks up to 5 POSTED
    items without evalScore (oldest-first), calls EvalJudge, then calls
    FeedItemEntity.recordEval(score, rationale) per item.

- 2 HttpEndpoints:
  * FeedEndpoint at /api with GET /feed, GET /feed/{id}, POST /feed/{id}/override
    (body {decidedBy, targetChannel}), POST /feed/{id}/suppress (body {decidedBy}),
    GET /feed/sse, and metadata endpoints serving YAML/MD from
    src/main/resources/metadata/. Override writes ReviewOverridden + NotifyRequested;
    suppress writes ReviewOverridden (suppress=true) + ItemSuppressed.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- FeedTasks.java declaring two Task<R> constants: SUMMARIZE (SummaryResult),
  EVAL (EvalResult).
- Domain records FeedItem, SummaryResult, ClassificationResult, SlackPost, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9511 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/feed-items.jsonl with 8 canned feed item lines
  covering NOTIFY, SUPPRESS, and REVIEW classifications.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail before-tool-call
  channel-scope, E1 eval-periodic performance-monitor. Matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root with decisions.authority_level = notify-only,
  oversight.human_in_loop = false (for NOTIFY/SUPPRESS path; true only for REVIEW path),
  failure.failure_modes including "off-channel-post" and "summary-quality-degradation";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/summary-agent.md, prompts/classifier-agent.md, prompts/eval-judge.md loaded
  as agent system prompts.
- README.md at the project root: title "Akka Sample: Feed Monitor", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column
  layout (left = live feed list with classification chips and eval score chips; right =
  selected item detail with AI summary, extracted topics, key quote, and Override/Suppress
  buttons for PENDING_REVIEW items). Browser title exactly:
  <title>Akka Sample: Feed Monitor</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.
- Same key-sourcing rules apply for SLACK_BOT_TOKEN: record only the reference,
  never the value.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    summary-agent.json — 6–8 SummaryResult entries for diverse topics
      (AI regulation, enterprise software, open-source tooling, market reports).
      Each entry has a 2–4 sentence summary, up to 5 topics, and a keyQuote
      that is a verbatim snippet. No invented URLs or author names.
    classifier-agent.json — 8–10 ClassificationResult entries spanning
      NOTIFY (high-signal research items), SUPPRESS (marketing copy, press
      releases with no novel content), and REVIEW (ambiguous or off-topic
      items). Each entry includes confidence and a one-sentence reason.
      Defaults to REVIEW under uncertainty. targetChannel is "#research-intel"
      for NOTIFY items and null for SUPPRESS/REVIEW.
    eval-judge.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric (summary quality / classification accuracy /
      channel appropriateness).
- A MockModelProvider.seedFor(itemId) helper makes per-item selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- SummaryAgent and ClassifierAgent are typed Agents, NOT AutonomousAgents.
- EvalJudge is an AutonomousAgent — never silently downgraded to Agent.
- The before-tool-call guardrail runs INSIDE SlackNotifier Consumer before
  the postToSlack stub is invoked — not inside an Agent's prompt.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
