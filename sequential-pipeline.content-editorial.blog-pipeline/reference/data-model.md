# Data model — AI Blog Writer Pipeline with Ollama

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Reference` | `source` | `String` | no | Short name of the originating reference. |
| | `url` | `String` | no | Canonical URL of the reference entry. |
| | `summary` | `String` | no | 1–2 sentence description of the reference. |
| | `fetchedAt` | `Instant` | no | When the RESEARCH phase recorded the reference. |
| `ResearchNotes` | `references` | `List<Reference>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `topicSummary` | `String` | no | 1-sentence summary of the topic from gathered references. |
| | `researchedAt` | `Instant` | no | When the RESEARCH task returned. |
| `OutlineSection` | `sectionId` | `String` | no | Slug. MUST be unique within an `Outline`. |
| | `heading` | `String` | no | Human-readable section heading. |
| | `keyPoints` | `List<String>` | no | 2–4 key points for this section. |
| `Outline` | `title` | `String` | no | Proposed post title. |
| | `sections` | `List<OutlineSection>` | no | Possibly empty. |
| | `outlinedAt` | `Instant` | no | When the OUTLINE task returned. |
| `DraftSection` | `sectionId` | `String` | no | MUST equal an `OutlineSection.sectionId` from the upstream `Outline`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 3–5 sentences of prose. |
| `Draft` | `title` | `String` | no | Post title. |
| | `introduction` | `String` | no | 2–3 sentence introduction. |
| | `sections` | `List<DraftSection>` | no | `sections.size() == outline.sections.size()`. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `ReadabilityScore` | `fleschKincaid` | `int` | no | Flesch-Kincaid readability score (0–100). |
| | `toneLabel` | `String` | no | E.g. `"technical-neutral"`, `"conversational"`, `"formal"`. |
| `EditedSection` | `sectionId` | `String` | no | MUST equal a `DraftSection.sectionId`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | Revised prose after tone adjustments. |
| | `readability` | `ReadabilityScore` | no | Readability score for this section. |
| `EditedDraft` | `title` | `String` | no | Post title (may be revised from Draft). |
| | `introduction` | `String` | no | Revised introduction. |
| | `sections` | `List<EditedSection>` | no | `sections.size() == draft.sections.size()`. |
| | `editedAt` | `Instant` | no | When the EDIT task returned. |
| `PostReference` | `label` | `String` | no | Display label (typically the reference source name). |
| | `url` | `String` | no | MUST equal a `Reference.url` from the upstream `ResearchNotes`. |
| `PostSection` | `sectionId` | `String` | no | MUST equal an `EditedSection.sectionId`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | Final prose (may include code fences for `tutorial` type). |
| | `references` | `List<PostReference>` | no | At least one reference per section (policy rule). |
| `Post` | `title` | `String` | no | Final post title. |
| | `summary` | `String` | no | 1–2 sentence summary. |
| | `postType` | `String` | no | `"tutorial"`, `"explainer"`, or `"opinion"`. |
| | `sections` | `List<PostSection>` | no | `sections.size() == outline.sections.size()`. |
| | `publishedAt` | `Instant` | no | When the PUBLISH task returned. |
| `PolicyCheckResult` | `passed` | `boolean` | no | Whether the guardrail accepted the prose. |
| | `reason` | `String` | no | "All checks passed." or structured violation reason. |
| | `checkedAt` | `Instant` | no | When `ContentPolicyGuardrail` ran. |
| `GuardrailBlockRecord` | `phase` | `String` | no | `RESEARCH` / `OUTLINE` / `DRAFT` / `EDIT` / `PUBLISH`. |
| | `reason` | `String` | no | Structured reason from `ContentPolicyGuardrail`. |
| | `checkedAt` | `Instant` | no | When the guardrail blocked. |
| `PostRecord` (entity state) | `postId` | `String` | no | — |
| | `topic` | `Optional<String>` | yes | Populated after `PostCreated`. |
| | `postType` | `Optional<String>` | yes | Populated after `PostCreated`. |
| | `research` | `Optional<ResearchNotes>` | yes | Populated after `ResearchCollected`. |
| | `outline` | `Optional<Outline>` | yes | Populated after `OutlineProduced`. |
| | `draft` | `Optional<Draft>` | yes | Populated after `DraftWritten`. |
| | `editedDraft` | `Optional<EditedDraft>` | yes | Populated after `EditApplied`. |
| | `post` | `Optional<Post>` | yes | Populated after `PostPublished`. |
| | `lastPolicyCheck` | `Optional<PolicyCheckResult>` | yes | Updated on every `ContentCleared` or `GuardrailBlocked` event. |
| | `status` | `PostStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PostCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailBlocks` | `List<GuardrailBlockRecord>` | no | Appended on every `GuardrailBlocked` event; empty on the happy path. |

Every nullable lifecycle field on `PostRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PostStatus`: `CREATED`, `RESEARCHING`, `RESEARCHED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `EDITING`, `EDITED`, `PUBLISHING`, `PUBLISHED`, `FAILED`.

## Events (`PostEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PostCreated` | `topic: String, postType: String` | → CREATED |
| `ResearchStarted` | — | → RESEARCHING |
| `ResearchCollected` | `research: ResearchNotes` | → RESEARCHED |
| `OutlineStarted` | — | → OUTLINING |
| `OutlineProduced` | `outline: Outline` | → OUTLINED |
| `DraftStarted` | — | → DRAFTING |
| `DraftWritten` | `draft: Draft` | → DRAFTED |
| `EditStarted` | — | → EDITING |
| `EditApplied` | `editedDraft: EditedDraft` | → EDITED |
| `PublishStarted` | — | → PUBLISHING |
| `ContentCleared` | `phase: String, checkedAt: Instant` | no status change (audit-only) |
| `PostPublished` | `post: Post` | → PUBLISHED (terminal happy) |
| `GuardrailBlocked` | `phase, reason, checkedAt` | no status change (audit-only) |
| `PostFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PostRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailBlocks = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PostRow` mirrors `PostRecord` exactly. The UI fetches the full row via `GET /api/posts/{id}` and streams updates via `GET /api/posts/sse`.

The view declares ONE query: `getAllPosts: SELECT * AS posts FROM post_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`BlogTasks.java`)

```java
public final class BlogTasks {
  public static final Task<ResearchNotes> RESEARCH_TOPIC = Task
      .name("Research topic")
      .description("Gather reference material about a topic by calling searchReferences and fetchSummary")
      .resultConformsTo(ResearchNotes.class);

  public static final Task<Outline> OUTLINE_POST = Task
      .name("Outline post")
      .description("Structure the post into sections with key points by calling structureSections and expandKeyPoints")
      .resultConformsTo(Outline.class);

  public static final Task<Draft> DRAFT_POST = Task
      .name("Draft post")
      .description("Write prose for each section by calling writeParagraph and composeTile")
      .resultConformsTo(Draft.class);

  public static final Task<EditedDraft> EDIT_POST = Task
      .name("Edit post")
      .description("Apply tone adjustments and check readability by calling applyToneAdjustments and checkReadability")
      .resultConformsTo(EditedDraft.class);

  public static final Task<Post> PUBLISH_POST = Task
      .name("Publish post")
      .description("Format the final post and collect references by calling formatPost and collectReferences")
      .resultConformsTo(Post.class);

  private BlogTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Policy-checked tools

`ContentPolicyGuardrail` registers on `BlogWriterAgent` via the `before-agent-response` hook. It checks the completed prose from each task response against three rules (prohibited-pattern scan, originality check, post-type alignment) before the workflow writes the result event onto the entity. See `eval-matrix.yaml` G1 for the full accept matrix.
