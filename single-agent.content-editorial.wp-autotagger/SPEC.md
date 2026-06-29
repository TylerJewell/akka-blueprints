# SPEC — wp-autotagger

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WP Autotagger.
**One-line pitch:** A user submits a WordPress post URL; one AI agent reads the post body (passed as a task attachment, never as inline prompt text) and returns a typed `TagProposal` — an ordered list of SEO tags — which is then applied to the post via the WordPress REST API.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One `TagProposerAgent` (AutonomousAgent) carries the entire tag-generation decision; the surrounding components only prepare its input and govern its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** (`TagProposalGuardrail`) runs before the agent's `applyTags` tool call executes. It confirms: the tag list is not empty, the tag count is within the configured maximum (default 10), every tag string is non-empty and contains no special characters, and the `postId` in the tool call matches the active tagging job. A rejected tool call returns a structured error to the agent loop so the task retries within its iteration budget.

The blueprint shows that even a single write-capable agent — one whose outputs flow directly into a CMS — can be governed without a second agent or an external review service. One guardrail at the tool-call boundary is the right cut.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a **Post URL** in the input field (or picks one of three seeded examples — a technology how-to post, a product-review post, a news summary post).
2. The user configures optional settings: **Max tags** (1–15, default 10), **Tag style** dropdown (descriptive / keyword-dense / mixed), and **Submitted by** text input.
3. The user clicks **Submit for tagging**. The UI POSTs to `/api/posts` and receives a `jobId`.
4. The card appears in the live list in `INGESTED` state. Within ~1 s, it transitions to `BODY_FETCHED` — the full post body is visible in the card detail, truncated to 500 chars.
5. Within ~10–30 s, the workflow's `tagStep` completes. The card transitions to `TAGGING` then `TAGS_APPLIED`. The tag proposal appears: a list of tag chips with their confidence scores, and the `tagStyle` used.
6. The user can submit another post URL; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PostEndpoint` | `HttpEndpoint` | `/api/posts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PostEntity`, `PostView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PostEntity` | `EventSourcedEntity` | Per-post tagging lifecycle: ingested → body-fetched → tagging → tags-applied. Source of truth. | `PostEndpoint`, `WpApiConsumer`, `TaggingWorkflow` | `PostView` |
| `WpApiConsumer` | `Consumer` | Subscribes to `PostIngested` events; fetches post body from stub WP API; calls `PostEntity.attachBody`. | `PostEntity` events | `PostEntity` |
| `TaggingWorkflow` | `Workflow` | One workflow per job. Steps: `awaitBodyStep` → `tagStep` → error. | started by `WpApiConsumer` once body is fetched | `TagProposerAgent`, `PostEntity` |
| `TagProposerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the tag configuration in the task definition and the post body as a task attachment; uses the `applyTags` tool to write tags back; returns `TagProposal`. | invoked by `TaggingWorkflow` | returns tag proposal; calls applyTags tool |
| `TagProposalGuardrail` | supporting class | before-tool-call hook on `TagProposerAgent`. Validates every `applyTags` tool call before it executes. | `TagProposerAgent` | passes or rejects |
| `WpApiStub` | supporting class | In-process stub of the WordPress REST API. Receives `applyTags` calls; records applied tags on the entity. | `TagProposerAgent` tool call | `PostEntity` |
| `PostView` | `View` | Read model: one row per tagging job for the UI. | `PostEntity` events | `PostEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TaggingConfig(
    int maxTags,
    TagStyle tagStyle,
    String submittedBy
) {}
enum TagStyle { DESCRIPTIVE, KEYWORD_DENSE, MIXED }

record PostRequest(
    String jobId,
    String postUrl,
    TaggingConfig config,
    Instant submittedAt
) {}

record PostBody(
    String postId,
    String title,
    String body,
    String author,
    Instant publishedAt
) {}

record TagEntry(
    String tag,
    double confidence     // 0.0–1.0
) {}

record TagProposal(
    List<TagEntry> tags,
    TagStyle tagStyle,
    String rationale,     // 1–2 sentences
    Instant proposedAt
) {}

record Post(
    String jobId,
    Optional<PostRequest> request,
    Optional<PostBody> body,
    Optional<TagProposal> proposal,
    PostStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PostStatus {
    INGESTED, BODY_FETCHED, TAGGING, TAGS_APPLIED, FAILED
}
```

Events on `PostEntity`: `PostIngested`, `PostBodyFetched`, `TaggingStarted`, `TagsApplied`, `PostFailed`.

Every nullable lifecycle field on the `Post` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/posts` — body `{ postUrl, config: { maxTags, tagStyle, submittedBy } }` → `{ jobId }`.
- `GET /api/posts` — list all tagging jobs, newest-first.
- `GET /api/posts/{id}` — one tagging job.
- `GET /api/posts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: WP Autotagger</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted tagging jobs (status pill + tag count + age) and a right pane with the selected job's detail — post URL, post body preview, tagging config, tag chips with confidence scores, and rationale.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs before every `applyTags` tool call inside `TagProposerAgent`. Asserts (1) the `postId` in the tool call matches the active job's post ID — scope check, (2) the tag list is not empty, (3) the tag count does not exceed `config.maxTags`, and (4) every tag string is non-empty and contains only alphanumeric characters, hyphens, and spaces. On failure, returns a structured `tool-call-rejected` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `TagProposerAgent` → `prompts/tag-proposer.md`. The single decision-making LLM. System prompt instructs it to read the attached post body, infer SEO-relevant tags aligned with the requested tag style, and call the `applyTags` tool with the final list.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the tech post seed; within 30 s the tag proposal appears with between 3 and `maxTags` tags, each with a confidence score, and an `TAGS_APPLIED` status.
2. **J2** — The agent's first `applyTags` tool call includes more tags than `maxTags` — the `before-tool-call` guardrail rejects it; the agent revises and re-calls with a trimmed list; the UI displays only the accepted proposal.
3. **J3** — The agent's first `applyTags` call targets a `postId` that does not match the active job — the guardrail rejects it as a scope violation; the agent corrects the `postId` and re-calls; the correct post receives the tags.
4. **J4** — A post body containing a WP application-password token is submitted; the LLM call log shows only `[REDACTED-TOKEN]`; the entity's `body.body` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named wp-autotagger demonstrating the single-agent × content-editorial cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-content-editorial-wp-autotagger. Java package
io.akka.samples.autotagblogpostsonwordpress. Akka 3.6.0. HTTP port 9796.

Components to wire (exactly):

- 1 AutonomousAgent TagProposerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/tag-proposer.md>) and
  .capability(TaskAcceptance.of(PROPOSE_TAGS).maxIterationsPerTask(3)).
  The task receives the tagging config (max tags, tag style) in its instruction text and the
  post body as a task ATTACHMENT (NOT as inline prompt text —
  Akka's TaskDef.attachment(name, contentBytes) is the canonical call). The agent has one
  tool: applyTags(postId: String, tags: List<TagEntry>) → TagProposal. The tool is
  registered via the agent's tool-configuration block. Output: TagProposal{tags:
  List<TagEntry>, tagStyle: TagStyle, rationale: String, proposedAt: Instant}. The agent
  is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered
  via the agent's guardrail-configuration block. On guardrail rejection the agent loop
  retries within its 3-iteration budget.

- 1 Workflow TaggingWorkflow per jobId with two steps plus error:
  * awaitBodyStep — polls PostEntity.getPost every 1 s; on post.body().isPresent() advances
    to tagStep. WorkflowSettings.stepTimeout 15 s (fetcher is in-process and fast).
  * tagStep — emits TaggingStarted, then calls componentClient.forAutonomousAgent(
    TagProposerAgent.class, "tagger-" + jobId).runSingleTask(
      TaskDef.instructions(formatConfig(post.request.config))
        .attachment("post.txt", post.body.body.getBytes())
    ) — returns a taskId, then forTask(taskId).result(PROPOSE_TAGS) to fetch the proposal.
    On success calls PostEntity.applyTags(proposal). WorkflowSettings.stepTimeout 60 s
    with defaultStepRecovery maxRetries(2).failoverTo(TaggingWorkflow::error).
  * error step transitions the entity to FAILED. WorkflowSettings.stepTimeout 5 s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PostEntity (one per jobId). State Post{jobId: String,
  request: Optional<PostRequest>, body: Optional<PostBody>,
  proposal: Optional<TagProposal>, status: PostStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. PostStatus enum: INGESTED, BODY_FETCHED, TAGGING,
  TAGS_APPLIED, FAILED. Events: PostIngested{request}, PostBodyFetched{body},
  TaggingStarted{}, TagsApplied{proposal}, PostFailed{reason}. Commands: ingest,
  attachBody, markTagging, applyTags, fail, getPost. emptyState() returns Post.initial("")
  with no commandContext() reference (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer WpApiConsumer subscribed to PostEntity events; on PostIngested calls the
  in-process WpApiStub to fetch the post body for the URL in request.postUrl, builds
  PostBody{postId, title, body, author, publishedAt}, then calls
  PostEntity.attachBody(postBody). After attachBody lands, the same Consumer starts a
  TaggingWorkflow with id = "tagging-" + jobId.

- 1 supporting class WpApiStub — in-process stub of the WordPress REST API. Reads from
  src/main/resources/sample-events/seed-posts.jsonl to resolve URLs to PostBody records.
  Returns a placeholder PostBody for any URL not in the seed file. Also receives
  applyTags(postId, tags) calls from the agent tool and records the applied tags by
  calling PostEntity.applyTags(proposal).

- 1 View PostView with row type PostRow (mirrors Post minus body.body — the audit log
  keeps the full body; the view holds a 500-char truncated preview for the UI).
  Table updater consumes PostEntity events. ONE query getAllPosts:
  SELECT * AS posts FROM post_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PostEndpoint at /api with POST /posts (body {postUrl, config: {maxTags, tagStyle,
    submittedBy}}; mints jobId; calls PostEntity.ingest; returns {jobId}), GET /posts
    (list from getAllPosts, sorted newest-first), GET /posts/{id} (one row), GET
    /posts/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- TaggingTasks.java declaring one Task<R> constant: PROPOSE_TAGS = Task.name("Propose
  tags").description("Read the attached post body and produce a TagProposal with SEO-
  relevant tags").resultConformsTo(TagProposal.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TaggingConfig, TagStyle, PostRequest, PostBody, TagEntry, TagProposal,
  Post, PostStatus.

- TagProposalGuardrail.java implementing the before-tool-call hook. Reads the candidate
  applyTags tool call, runs the four checks listed in eval-matrix.yaml G1, and either
  passes through or returns Guardrail.reject(<structured-error>) to force a retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9796 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  TagProposerAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/sample-events/seed-posts.jsonl with 3 seeded post examples: a
  technology how-to post (~800 words, covering a programming topic), a product-review
  post (~600 words, covering a consumer gadget), and a news summary post (~400 words,
  covering a current-events topic). Each contains 1–2 plausible credential-like tokens
  so the body-redaction path has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (post
  bodies are not personal data), decisions.authority_level = automated (tags are
  applied without human approval), oversight.human_in_loop = false,
  failure.failure_modes including "off-topic-tags", "tag-count-overflow",
  "wrong-post-id", "credential-leakage-via-llm"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/tag-proposer.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: WP Autotagger", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of tagging job cards; right = selected job detail with post URL,
  body preview, tagging config, tag chips with confidence scores, rationale).
  Browser title exactly: <title>Akka Sample: WP Autotagger</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using
        the matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(jobId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    propose-tags.json — 8 TagProposal entries covering all three TagStyle values and
    ranging from 3 to 10 tags each. Each entry has a rationale sentence and a
    `tags` array where each TagEntry has a non-empty `tag` string and a `confidence`
    between 0.6 and 1.0. Plus 2 deliberately MALFORMED entries: one with a tag count
    exceeding the default maxTags=10, one with a `postId` that does not match the
    active job. The guardrail blocks both, exercising the retry path. The mock selects
    a malformed entry on the FIRST iteration of every 3rd job (modulo seed) so J2/J3
    are reproducible.
- A MockModelProvider.seedFor(jobId) helper makes per-job selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TagProposerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TaggingTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (tagStep 60 s, awaitBodyStep 15 s, error 5 s).
- Lesson 6: every nullable lifecycle field on the Post row record is Optional<T>.
- Lesson 7: TaggingTasks.java with PROPOSE_TAGS = Task.name(...).description(...)
  .resultConformsTo(TagProposal.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9796 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative or UI copy.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: exactly ONE AutonomousAgent (TagProposerAgent). No eval
  agent. No secondary LLM call.
- The post body is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated tagStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external post-hoc check. Lesson 1's AutonomousAgent contract
  is the authoritative reference for how the hook is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
