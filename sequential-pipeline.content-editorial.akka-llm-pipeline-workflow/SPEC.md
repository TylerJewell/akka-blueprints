# SPEC — akka-llm-pipeline-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Deterministic Workflow with LLM Activities.
**One-line pitch:** A user submits a topic; a durable `ContentWorkflow` calls two LLM activities in sequence — **OUTLINE** a structured post plan, then **DRAFT** a finished blog post from that plan — with each activity's output schema-validated and the final draft reviewed by an editorial guardrail before the post is published.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern expressed as a pure Akka Workflow with two LLM-backed activities — no autonomous agent, no tool loop. `OutlineActivity` and `DraftActivity` are each a single, typed, durable LLM call. The pattern's defining property is the **typed handoff**: `outlineStep` writes a `PostOutline` onto `PostEntity`; `draftStep` reads that outline from the entity and passes it as the sole context for the draft call. The draft LLM never sees the raw topic — the structured outline is the only information path between activities.

Two governance mechanisms are wired around the workflow:

- An **`after-llm-response` guardrail** (`SchemaGuardrail`) runs immediately after `OutlineActivity` returns, before the workflow advances to `draftStep`. It validates that the `PostOutline` satisfies all required structural constraints: `title` is non-blank, `sections` is non-empty, every `section.heading` is non-blank, and `wordTarget > 0`. A failing outline is not passed to the draft step; the workflow transitions to `FAILED` and records a structured `ValidationFailed` event so the reason is visible in the UI.
- A **`before-agent-response` guardrail** (`EditorialGuardrail`) runs after `DraftActivity` returns and before the workflow marks the post `READY`. It applies rule-based editorial checks: the draft's word count is within ±20% of the declared `wordTarget`, no prohibited topic phrases appear in the body, and every section cited in the outline has a matching section in the draft. Posts that fail any check land in `FLAGGED` state; posts that pass all checks advance to `READY`.

The blueprint shows that a sequential pipeline of LLM activities — not agents — is sufficient when there is no need for a tool-calling loop. The two guardrails enforce the output quality contract at the two points where the pipeline is most vulnerable: the structured-output boundary after the first LLM call, and the user-facing boundary before the second LLM call's output is published.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **topic** into the input (or picks one of three seeded topics — `The future of edge computing`, `Zero-trust security in practice`, `Building resilient distributed systems`).
2. The user optionally sets a **word target** (default 600). The user clicks **Generate post**. The UI POSTs to `/api/posts` and receives a `postId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `OUTLINING` — the workflow has started `outlineStep`.
4. Within ~10–15 s the card reaches `DRAFTING`. The right pane shows the `PostOutline` — title, word target, and the list of sections with their headings and key-point bullets. The `after-llm-response` guardrail passed.
5. Within ~15–25 s more the card reaches `REVIEWING`. The `DraftActivity` returned a `BlogPost`; the `EditorialGuardrail` is now running.
6. Within ~2 s the card reaches `READY` (guardrail passed) or `FLAGGED` (guardrail caught a policy violation). The right pane shows the full `BlogPost` — title, introduction, body sections, conclusion — plus an editorial-review chip (PASSED or FLAGGED with reason).
7. The user can submit another topic; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PostEndpoint` | `HttpEndpoint` | `/api/posts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PostEntity`, `PostView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PostEntity` | `EventSourcedEntity` | Per-post lifecycle: created → outlining → outlined → drafting → drafted → reviewing → ready / flagged / failed. Source of truth. | `PostEndpoint`, `ContentWorkflow` | `PostView` |
| `ContentWorkflow` | `Workflow` | One workflow per post. Steps: `outlineStep` → `draftStep` → `reviewStep`. A fourth `failStep` handles errors. Each step calls the appropriate activity, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `PostEndpoint` after `CREATED` | `OutlineActivity`, `DraftActivity`, `SchemaGuardrail`, `EditorialGuardrail`, `PostEntity` |
| `OutlineActivity` | `ActivityCaller` (typed LLM call) | Single LLM call: takes `(topic, wordTarget)`, returns `PostOutline`. Result is checked by `SchemaGuardrail` before workflow advances. | called by `ContentWorkflow.outlineStep` | returns `PostOutline` |
| `DraftActivity` | `ActivityCaller` (typed LLM call) | Single LLM call: takes `PostOutline`, returns `BlogPost`. Result is checked by `EditorialGuardrail` before workflow marks post `READY`. | called by `ContentWorkflow.draftStep` | returns `BlogPost` |
| `SchemaGuardrail` | `after-llm-response` guardrail (registered on `OutlineActivity`) | Validates required fields of the returned `PostOutline`. On failure, calls `PostEntity.recordValidationFailure` and signals the workflow to transition to `FAILED`. | every `OutlineActivity` response | accept / structured-reject |
| `EditorialGuardrail` | `before-agent-response` guardrail (registered on `DraftActivity`) | Applies word-count, prohibited-topic, and section-coverage rules to the returned `BlogPost`. On failure, calls `PostEntity.recordEditorialFlag`; workflow transitions to `FLAGGED`. On pass, workflow transitions to `READY`. | every `DraftActivity` response | accept / structured-flag |
| `PostView` | `View` | Read model: one row per post for the UI. | `PostEntity` events | `PostEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record OutlineSection(String heading, List<String> keyPoints) {}

record PostOutline(
    String title,
    int wordTarget,
    List<OutlineSection> sections,
    Instant outlinedAt
) {}

record DraftSection(String heading, String body) {}

record BlogPost(
    String title,
    String introduction,
    List<DraftSection> sections,
    String conclusion,
    int wordCount,
    Instant draftedAt
) {}

record EditorialReview(
    boolean passed,
    String reason,    // "all checks passed" on happy path; structured reason on flag
    Instant reviewedAt
) {}

record ValidationFailure(
    String field,     // which field failed: "title", "sections", "wordTarget"
    String reason,
    Instant failedAt
) {}

record PostRecord(
    String postId,
    Optional<String> topic,
    Optional<Integer> wordTarget,
    Optional<PostOutline> outline,
    Optional<BlogPost> draft,
    Optional<EditorialReview> review,
    Optional<ValidationFailure> validationFailure,
    PostStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PostStatus {
    CREATED, OUTLINING, OUTLINED, DRAFTING, DRAFTED,
    REVIEWING, READY, FLAGGED, FAILED
}
```

Events on `PostEntity`: `PostCreated`, `OutlineStarted`, `OutlineProduced`, `DraftStarted`, `DraftProduced`, `ReviewStarted`, `PostApproved`, `PostFlagged`, `ValidationFailed`, `PostFailed`.

Every nullable lifecycle field on the `PostRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/posts` — body `{ topic, wordTarget? }` → `{ postId }`.
- `GET /api/posts` — list all posts, newest-first.
- `GET /api/posts/{id}` — one post.
- `GET /api/posts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Deterministic Workflow with LLM Activities</title>`.

The App UI tab is a two-column layout: a left rail with the live list of posts (status pill + topic + age) and a right pane with the selected post's detail — topic, word target, outline sections table, draft body, editorial-review chip, and a validation-failure strip if the outline was rejected.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — `after-llm-response` guardrail (schema-validation)**: `SchemaGuardrail` is registered on `OutlineActivity` and runs immediately after the outline LLM call returns, before the workflow advances to `draftStep`. It validates four structural constraints: `title` is non-blank, `sections` is non-empty (at least one `OutlineSection`), every `section.heading` is non-blank, and `wordTarget > 0`. On failure, the guardrail returns a structured `validation-failed` error carrying the failing field name and reason. The workflow records `ValidationFailed{field, reason}` on the entity and transitions the post to `FAILED`. The validation failure is visible on the UI card so the user understands what went wrong. This check catches malformed LLM outputs before they silently propagate downstream as bad context for the draft call.
- **H2 — `before-agent-response` guardrail (editorial review)**: `EditorialGuardrail` is registered on `DraftActivity` and runs after the draft LLM call returns and before the workflow commits the post as `READY`. Three editorial rules, each independently checked: (1) word-count constraint — `BlogPost.wordCount` is within ±20% of `PostOutline.wordTarget`; (2) prohibited-topic check — none of the prohibited-phrase list appears in `BlogPost.introduction` or any `DraftSection.body`; (3) section-coverage check — every `OutlineSection.heading` from the upstream `PostOutline` has a matching `DraftSection.heading` in the `BlogPost` (case-insensitive). On any failure, the guardrail returns a structured `editorial-flag` carrying the failing rule and a reason sentence. The workflow writes `PostFlagged{reason}` and transitions to `FLAGGED`. On all three passing, the workflow writes `PostApproved` and transitions to `READY`. The UI highlights `FLAGGED` cards in amber.

## 9. Agent prompts

- `OutlineActivity` → `prompts/outline-activity.md`. The LLM prompt for the outline activity. Instructs the model to produce a structured `PostOutline` from a topic and word target.
- `DraftActivity` → `prompts/draft-activity.md`. The LLM prompt for the draft activity. Instructs the model to expand a `PostOutline` into a finished `BlogPost`, section by section, staying within the word target.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded topic `The future of edge computing` with default word target 600; within 60 s the post reaches `READY` with a non-empty outline (≥ 2 sections), a non-empty draft matching the outline structure, and an editorial review chip showing `PASSED`.
2. **J2** — The mock-LLM outline for a given briefing contains an empty `sections` list. `SchemaGuardrail` catches it at the `after-llm-response` hook; `ValidationFailed{field: "sections", reason: "..."}` lands on the entity; the post transitions to `FAILED`. The UI card shows the structured failure reason. The draft activity is never called.
3. **J3** — The mock-LLM draft contains a prohibited-topic phrase. `EditorialGuardrail` catches it at the `before-agent-response` hook; `PostFlagged{reason: "prohibited-topic: ..."}` lands on the entity; the post transitions to `FLAGGED`. The UI card highlights amber; the reader sees the flag reason.
4. **J4** — The outline produced by `outlineStep` is the only input to `draftStep`. The service log shows the `draftStep`'s LLM call context contains only the serialised `PostOutline`; the raw topic string does not appear in the draft call's input.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-llm-pipeline-workflow demonstrating the sequential-pipeline
x content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact sequential-pipeline-content-editorial-akka-llm-pipeline-workflow.
Java package io.akka.samples.deterministicworkflowwithllmactivities. Akka 3.6.0.
HTTP port 9854.

Components to wire (exactly):

- 1 Workflow ContentWorkflow per postId with four steps:
  * outlineStep — emits OutlineStarted on the entity, then calls a typed LLM activity
    (OutlineActivity) with the topic and wordTarget from the PostRecord. Receives a
    PostOutline. Runs SchemaGuardrail over the result; on failure, records ValidationFailed
    on the entity and transitions to failStep. On pass, writes PostEntity.recordOutline(
    outline). stepTimeout 60s.
  * draftStep — emits DraftStarted on the entity, then calls a typed LLM activity
    (DraftActivity) with the PostOutline read from the entity (NOT the original topic).
    Receives a BlogPost. Runs EditorialGuardrail over the result; on flag, records
    PostFlagged on the entity and transitions post to FLAGGED and ends. On pass, records
    PostApproved and transitions to READY and ends. stepTimeout 90s.
  * reviewStep — emits ReviewStarted; this is the step that invokes EditorialGuardrail
    if implemented as a separate step rather than inline in draftStep (either approach
    is acceptable; keep it consistent). stepTimeout 10s.
  * failStep — records PostFailed{reason} and ends. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(ContentWorkflow::failStep). The failStep writes PostFailed
  and ends.

- 1 EventSourcedEntity PostEntity (one per postId). State PostRecord{postId,
  topic: Optional<String>, wordTarget: Optional<Integer>, outline: Optional<PostOutline>,
  draft: Optional<BlogPost>, review: Optional<EditorialReview>,
  validationFailure: Optional<ValidationFailure>, status: PostStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. PostStatus enum: CREATED,
  OUTLINING, OUTLINED, DRAFTING, DRAFTED, REVIEWING, READY, FLAGGED, FAILED. Events:
  PostCreated{topic, wordTarget}, OutlineStarted, OutlineProduced{outline}, DraftStarted,
  DraftProduced{draft}, ReviewStarted, PostApproved{review}, PostFlagged{reason, review},
  ValidationFailed{field, reason}, PostFailed{reason}.
  Commands: create, startOutline, recordOutline, startDraft, recordDraft, startReview,
  approve, flag, recordValidationFailure, fail, getPost. emptyState() returns
  PostRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View PostView with row type PostRow that mirrors PostRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes PostEntity events. ONE query
  getAllPosts: SELECT * AS posts FROM post_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PostEndpoint at /api with POST /posts (body {topic, wordTarget?}; mints postId;
    calls PostEntity.create(topic, wordTarget); then starts ContentWorkflow with id
    "workflow-" + postId; returns {postId}), GET /posts (list from getAllPosts, sorted
    newest-first), GET /posts/{id} (one row), GET /posts/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion / helper classes:

- SchemaGuardrail.java — implements the after-llm-response hook. Reads the returned
  PostOutline (deserialised from the LLM response before the workflow sees it) and applies
  four checks: title non-blank, sections non-empty, every section.heading non-blank,
  wordTarget > 0. On any failure, returns Guardrail.reject("validation-failed: <field>
  <reason>") and calls PostEntity.recordValidationFailure(field, reason). On all passing,
  returns Guardrail.accept().

- EditorialGuardrail.java — implements the before-agent-response hook. Reads the returned
  BlogPost and applies three rules: (1) word count within ±20% of outline.wordTarget;
  (2) no prohibited phrase (case-insensitive list in src/main/resources/editorial-policy/
  prohibited-phrases.txt) appears in the introduction or any DraftSection.body;
  (3) every OutlineSection.heading from the upstream PostOutline has a matching
  DraftSection.heading in the BlogPost (case-insensitive). On any failure, returns
  Guardrail.flag("editorial-flag: <rule> <reason>") and calls
  PostEntity.flag(reason). On all passing, returns Guardrail.accept().

- OutlineActivity.java — the typed LLM call for the outline step. Loads
  prompts/outline-activity.md as the system prompt. Takes (topic: String,
  wordTarget: int). Returns PostOutline. Registered with SchemaGuardrail via the
  activity's guardrail-configuration block (after-llm-response).

- DraftActivity.java — the typed LLM call for the draft step. Loads
  prompts/draft-activity.md as the system prompt. Takes PostOutline. Returns BlogPost.
  Registered with EditorialGuardrail via the activity's guardrail-configuration block
  (before-agent-response).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9854 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/topics.jsonl with 5 seeded topic lines covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/editorial-policy/prohibited-phrases.txt — newline-delimited list of
  10 placeholder prohibited phrases used by EditorialGuardrail for deterministic
  offline testing.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, H2) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors —
  general domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = recommend-only, oversight.human_in_loop = true,
  operations.agent_count = 0 (no AutonomousAgent), operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "schema-invalid-outline",
  "editorial-policy-violation", "draft-word-count-out-of-bounds",
  "section-coverage-gap"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/outline-activity.md and prompts/draft-activity.md loaded as LLM prompts.

- README.md at the project root: title "Akka Sample: Deterministic Workflow with LLM
  Activities", prerequisites, generate-the-system, what-you-get, customise-before-
  generating, what-gets-validated, license. NO Configuration section. NO governance-
  mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of post cards; right = selected-post detail with topic header, outline
  sections table, draft body sections, editorial-review chip, validation-failure strip).
  Browser title exactly: <title>Akka Sample: Deterministic Workflow with LLM Activities
  </title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per activity (outline-activity and draft-activity).
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured
  key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-
  activity dispatch on the activity type id. Each branch reads src/main/resources/
  mock-responses/<activity-id>.json, picks one entry pseudo-randomly per call (seedFor(
  postId)), and deserialises into the activity's typed return.
- Per-activity mock-response shapes for THIS blueprint:
    outline-activity.json — 6 PostOutline entries, each with 2-4 OutlineSection items
      and valid wordTargets. Plus 1 deliberately INVALID entry with sections = [] (empty
      section list) — the SchemaGuardrail catches it; J2 verifies this. The invalid entry
      is selected on the FIRST run of every 4th post (modulo seed) so J2 is reproducible.
    draft-activity.json — 6 BlogPost entries paired one-to-one with the outline entries,
      each with matching DraftSection headings and wordCounts within ±20% of wordTarget.
      Plus 1 deliberately FLAGGED entry whose introduction contains a prohibited phrase
      from prohibited-phrases.txt — EditorialGuardrail catches it; J3 verifies this.
- A MockModelProvider.seedFor(postId) helper makes per-post selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: this blueprint uses NO AutonomousAgent. OutlineActivity and DraftActivity are
  typed LLM activities (ActivityCaller), not agents. If the Akka SDK does not have a
  dedicated ActivityCaller primitive, implement them as typed LLM calls within the
  Workflow steps using the ModelClient API directly. Do NOT silently introduce an
  AutonomousAgent.
- Lesson 4: every workflow step has an explicit stepTimeout (outlineStep 60s, draftStep
  90s, reviewStep 10s, failStep 5s).
- Lesson 6: every nullable lifecycle field on the PostRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...)
  or .isPresent().
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9854 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The no-agent invariant: there are ZERO AutonomousAgent instances. Both LLM calls are
  typed activities inside a Workflow. The guardrails are registered on activities, not
  on agents.
- The sequential-pipeline invariant: outlineStep writes PostOutline onto the entity;
  draftStep reads it from the entity and passes it as the sole LLM context. The raw
  topic is never passed to the draft activity — the typed handoff is the only information
  path between activities.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
