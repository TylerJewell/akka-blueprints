# SPEC — gitty

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Gitty.
**One-line pitch:** A maintainer submits a GitHub issue or pull-request URL; one AI agent reads the full thread (passed as a task attachment, never inlined into the prompt) and returns a structured draft reply the maintainer reviews and approves before posting.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the developer-tooling domain. One `ReplyDrafterAgent` (AutonomousAgent) authors the draft; every other component prepares its input or routes its output. Two governance mechanisms are wired around the agent:

- A **before-agent-response guardrail** runs on every turn of `ReplyDrafterAgent`. It checks that the draft reply is on-topic (references content from the thread), meets tone standards (no confrontational language, no sarcasm, no dismissals), and does not exceed the configured length ceiling. A draft that fails any check is rejected and causes the agent to retry within the same task's iteration budget.
- An **application-level HITL gate** holds every draft in a `PENDING_REVIEW` queue until the maintainer explicitly approves or discards it via the UI. The draft is never posted to GitHub by the system; the maintainer is the final decision-maker. The entity records `DraftApproved` or `DraftDiscarded` and the UI updates accordingly.

The blueprint demonstrates that a single-agent pattern can satisfy a meaningful governance bar: one structural guardrail and one human gate, with no second agent needed.

## 3. User-facing flows

The user opens the App UI tab.

1. The maintainer pastes a GitHub issue or pull-request URL into the **Thread URL** input (e.g., `https://github.com/akka/akka/issues/4321`), or picks one of three seeded examples. Optionally adds a brief note in the **Context** field (e.g., "We closed this in v3; be concise").
2. The maintainer clicks **Queue draft**. The UI POSTs to `/api/drafts` and receives a `draftId`.
3. The card appears in the live list in `QUEUED` state. Within ~1 s it transitions to `THREAD_FETCHED` — the fetched issue title and comment count are visible in the card detail.
4. Within ~10–30 s the workflow's `draftStep` completes. The card transitions to `DRAFTING` then `PENDING_REVIEW`. The draft reply text is visible in the card detail, along with the tone-check pass indicator.
5. The maintainer reads the draft, optionally edits it in the detail pane, then clicks **Approve** or **Discard**.
6. On **Approve**, the entity records `DraftApproved`; the card status changes to `APPROVED` and the draft text is shown with a copy button (deployers wire the actual GitHub POST separately). On **Discard**, status becomes `DISCARDED`.
7. The live list keeps all drafts visible; maintainers can submit new threads without clearing old ones.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DraftEndpoint` | `HttpEndpoint` | `/api/drafts/*` — queue, list, get, SSE, approve, discard; serves `/api/metadata/*`. | — | `DraftEntity`, `DraftView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DraftEntity` | `EventSourcedEntity` | Per-draft lifecycle: queued → thread-fetched → drafting → pending-review → approved/discarded/failed. Source of truth. | `DraftEndpoint`, `ThreadFetcher`, `DraftWorkflow` | `DraftView` |
| `ThreadFetcher` | `Consumer` | Subscribes to `ThreadQueued` events; calls the GitHub REST API to pull the issue/PR thread; calls `DraftEntity.attachThread`. | `DraftEntity` events | `DraftEntity` |
| `DraftWorkflow` | `Workflow` | One workflow per draftId. Steps: `awaitThreadStep` → `draftStep` → `pendingReviewStep`. | started by `ThreadFetcher` once thread is fetched | `ReplyDrafterAgent`, `DraftEntity` |
| `ReplyDrafterAgent` | `AutonomousAgent` | The one decision-making LLM. Receives context note in the task instructions and the full thread JSON as a task attachment; returns `DraftReply`. | invoked by `DraftWorkflow` | returns draft |
| `DraftView` | `View` | Read model: one row per draft for the UI. | `DraftEntity` events | `DraftEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ThreadRef(
    String url,
    String repoSlug,        // "owner/repo"
    long threadNumber,
    ThreadKind kind         // ISSUE or PULL_REQUEST
) {}
enum ThreadKind { ISSUE, PULL_REQUEST }

record QueueRequest(
    String draftId,
    ThreadRef thread,
    String contextNote,     // optional maintainer hint
    String queuedBy,
    Instant queuedAt
) {}

record FetchedThread(
    String title,
    String body,
    List<String> comments,  // chronological comment bodies
    int commentCount,
    String fetchedAt        // ISO-8601
) {}

record DraftReply(
    String text,
    String tone,            // e.g. "helpful", "closing", "asking-for-info"
    List<String> references,// issue/PR numbers or topics the draft cites
    Instant draftedAt
) {}

record MaintainerAction(
    String actorId,
    String editedText,      // non-null on Approve; null on Discard
    Instant actedAt
) {}

record Draft(
    String draftId,
    Optional<QueueRequest> request,
    Optional<FetchedThread> thread,
    Optional<DraftReply> draft,
    Optional<MaintainerAction> action,
    DraftStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DraftStatus {
    QUEUED, THREAD_FETCHED, DRAFTING, PENDING_REVIEW,
    APPROVED, DISCARDED, FAILED
}
```

Events on `DraftEntity`: `ThreadQueued`, `ThreadFetched`, `DraftingStarted`, `DraftProduced`, `DraftApproved`, `DraftDiscarded`, `DraftFailed`.

Every nullable lifecycle field on the `Draft` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/drafts` — body `{ url, contextNote, queuedBy }` → `{ draftId }`.
- `GET /api/drafts` — list all drafts, newest-first.
- `GET /api/drafts/{id}` — one draft.
- `GET /api/drafts/sse` — Server-Sent Events; one event per state transition.
- `POST /api/drafts/{id}/approve` — body `{ actorId, editedText }` → `204`.
- `POST /api/drafts/{id}/discard` — body `{ actorId }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Gitty</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted drafts (status pill + tone badge + age) and a right pane with the selected draft's detail — thread title + URL, fetched comment count, context note, the draft text (editable), and action buttons (Approve / Discard).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ReplyDrafterAgent`. Asserts that (1) the draft text references at least one concrete element from the fetched thread (title, a commenter's point, or the issue number), (2) the draft contains no confrontational phrases (curated blocklist), (3) the draft does not exceed 800 characters, and (4) the `tone` field is one of `{ helpful, closing, asking-for-info, acknowledging }`. On failure, returns a structured `invalid-draft` error to the agent loop naming which check failed; the agent retries within its 3-iteration budget.
- **H1 — application-level HITL**: every draft that passes G1 lands in `PENDING_REVIEW`. The UI presents the draft to the maintainer who reads, optionally edits, then explicitly approves or discards. The entity records `DraftApproved` (with the possibly-edited text) or `DraftDiscarded`. No draft is posted to GitHub by Gitty — that step is a deployer responsibility.

## 9. Agent prompts

- `ReplyDrafterAgent` → `prompts/reply-drafter.md`. The single decision-making LLM. System prompt instructs it to read the attached thread JSON, understand the issue or PR context, and write a concise, helpful reply on behalf of the project maintainer.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Maintainer queues a seeded issue URL; within 30 s a `PENDING_REVIEW` draft appears in the UI with a non-empty draft text and a `tone` value in the allowed set.
2. **J2** — The agent's first response fails the tone check (mock LLM path); the `before-agent-response` guardrail rejects it; the second iteration produces a compliant draft; the non-compliant text never appears in the entity log.
3. **J3** — Maintainer approves a draft; the entity transitions to `APPROVED`; the UI shows a copy button with the approved text. The approved text (edited or original) matches exactly what `DraftApproved.editedText` records.
4. **J4** — Maintainer discards a draft; status transitions to `DISCARDED`; the draft is removed from the PENDING_REVIEW section of the queue.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named gitty demonstrating the single-agent × dev-code cell. Maven group
io.akka.samples. Maven artifact single-agent-dev-code-gh-draft-replier.
Java package io.akka.samples.gitty. Akka 3.6.0. HTTP port 9510.

Components to wire (exactly):

- 1 AutonomousAgent ReplyDrafterAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/reply-drafter.md>) and
  .capability(TaskAcceptance.of(DRAFT_REPLY).maxIterationsPerTask(3)). The task receives
  the maintainer's context note as its instruction text and the full fetched thread JSON
  as a task ATTACHMENT (NOT as inline prompt text — use TaskDef.attachment("thread.json",
  threadBytes)). Output: DraftReply{text: String, tone: String, references: List<String>,
  draftedAt: Instant}. The agent is configured with a before-agent-response guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow DraftWorkflow per draftId with three steps:
  * awaitThreadStep — polls DraftEntity.getDraft every 1s; on draft.thread().isPresent()
    advances to draftStep. WorkflowSettings.stepTimeout 20s (fetcher is near-local).
  * draftStep — emits DraftingStarted, then calls componentClient.forAutonomousAgent(
    ReplyDrafterAgent.class, "drafter-" + draftId).runSingleTask(
      TaskDef.instructions(draft.request.contextNote())
        .attachment("thread.json", serializeThread(draft.thread.get()).getBytes())
    ) — returns a taskId, then forTask(taskId).result(DRAFT_REPLY) to fetch the draft.
    On success calls DraftEntity.recordDraft(draftReply). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(DraftWorkflow::error).
  * pendingReviewStep — records DraftProduced and waits; this step completes when
    DraftWorkflow.approveDraft(action) or DraftWorkflow.discardDraft(action) is called
    from DraftEndpoint. WorkflowSettings.stepTimeout 0 (no timeout — human gate).
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity DraftEntity (one per draftId). State Draft{draftId: String,
  request: Optional<QueueRequest>, thread: Optional<FetchedThread>,
  draft: Optional<DraftReply>, action: Optional<MaintainerAction>, status: DraftStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. DraftStatus enum: QUEUED,
  THREAD_FETCHED, DRAFTING, PENDING_REVIEW, APPROVED, DISCARDED, FAILED. Events:
  ThreadQueued{request}, ThreadFetched{thread}, DraftingStarted{}, DraftProduced{draft},
  DraftApproved{action}, DraftDiscarded{action}, DraftFailed{reason}. Commands: queue,
  attachThread, markDrafting, recordDraft, approveDraft, discardDraft, fail, getDraft.
  emptyState() returns Draft.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer ThreadFetcher subscribed to DraftEntity events; on ThreadQueued calls the
  GitHub REST API (GET /repos/{owner}/{repo}/issues/{number} and
  GET /repos/{owner}/{repo}/issues/{number}/comments) using the GITHUB_TOKEN env var.
  Assembles FetchedThread{title, body, comments (list of comment bodies), commentCount,
  fetchedAt}. Calls DraftEntity.attachThread(thread). After attachThread lands, the same
  Consumer starts a DraftWorkflow with id = "draft-" + draftId. The GITHUB_TOKEN is read
  from ${?GITHUB_TOKEN} in application.conf — never hardcoded.

- 1 View DraftView with row type DraftRow (mirrors Draft minus thread.comments — the full
  comment list is large; the UI fetches it on demand via GET /api/drafts/{id}). Table
  updater consumes DraftEntity events. ONE query getAllDrafts: SELECT * AS drafts FROM
  draft_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * DraftEndpoint at /api with POST /drafts (body {url, contextNote, queuedBy}; parses
    url into ThreadRef; mints draftId; calls DraftEntity.queue; returns {draftId}),
    GET /drafts (list from getAllDrafts, sorted newest-first), GET /drafts/{id} (one row),
    GET /drafts/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /drafts/{id}/approve (body {actorId, editedText}; calls
    DraftWorkflow.approveDraft; returns 204), POST /drafts/{id}/discard (body {actorId};
    calls DraftWorkflow.discardDraft; returns 204), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- DraftTasks.java declaring one Task<R> constant: DRAFT_REPLY = Task.name("Draft reply")
  .description("Read the attached GitHub thread and produce a DraftReply for the maintainer")
  .resultConformsTo(DraftReply.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ThreadRef, ThreadKind, QueueRequest, FetchedThread, DraftReply,
  MaintainerAction, Draft, DraftStatus.

- ToneGuardrail.java implementing the before-agent-response hook. Reads the candidate
  DraftReply from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>)
  to force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9510 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  ReplyDrafterAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern.
  GitHub token block: github.token = ${?GITHUB_TOKEN}.

- src/main/resources/sample-events/seeded-threads.jsonl with 3 seeded GitHub thread
  snapshots in the FetchedThread shape: one "feature request left unanswered for 90 days",
  one "bug report where the fix was merged but the issue wasn't closed", and one
  "first-time contributor asking for review feedback". Each has a title, body, 2–4
  comments, and commentCount.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with domain=dev-code, decisions.authority_level=
  recommend-only (draft is advisory), oversight.human_in_loop=true, failure.failure_modes
  including "off-topic-draft", "confrontational-tone", "hallucinated-thread-reference",
  "draft-posted-without-review"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/reply-drafter.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Gitty", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of draft cards; right = selected-draft detail with thread info,
  context note, draft text, approve/discard buttons). Browser title exactly:
  <title>Akka Sample: Gitty</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY and GITHUB_TOKEN.
  If exactly one model-provider key is set, default application.conf's model-provider
  to match and proceed silently.
- If no model-provider key is set, ask the user how to source the key, offering five
  options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct DraftReply objects per task (see Mock LLM provider block below).
        Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- If GITHUB_TOKEN is not set, ask separately for the token using the same five options.
  In mock mode (option a), generate a MockThreadFetcher that returns seeded thread JSON
  from src/main/resources/sample-events/seeded-threads.jsonl without calling GitHub.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface. Reads
  src/main/resources/mock-responses/draft-reply.json, picks one entry pseudo-randomly
  per call (seedFor(draftId)), and deserialises into DraftReply.
- draft-reply.json — 6 DraftReply entries covering all four tone values plus 2
  deliberately NON-COMPLIANT entries: one with a confrontational phrase ("that's not
  how this works") and one with a tone value outside the enum ("sarcastic"). The mock
  selects a non-compliant entry on the FIRST iteration of every 3rd draft (modulo seed)
  so J2 is reproducible.
- MockModelProvider.seedFor(draftId) makes per-draft selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ReplyDrafterAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DraftTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitThreadStep 20s,
  draftStep 60s, pendingReviewStep 0 = no timeout for human gate, error 5s).
- Lesson 6: every nullable lifecycle field on the Draft row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: DraftTasks.java with DRAFT_REPLY = Task.name(...).description(...)
  .resultConformsTo(DraftReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9510 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Requires GitHub token" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (ReplyDrafterAgent). The GitHub
  thread fetch is a Consumer, not a second agent. No LLM call happens in ThreadFetcher.
- The thread is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated draftStep uses TaskDef.attachment("thread.json", ...) and not
  string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
- The HITL gate is advisory: DraftEntity records the maintainer's decision but does NOT
  call the GitHub API to post the comment. That step is explicitly out-of-scope and must
  not be generated.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid LLM env vars plus `GITHUB_TOKEN`; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
