# Data model — video-to-blog-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Chapter` | `title` | `String` | no | Short descriptive label for the chapter. |
| | `startSeconds` | `int` | no | Second offset where the chapter begins. |
| | `endSeconds` | `int` | no | Second offset where the chapter ends. |
| `Transcript` | `videoUrl` | `String` | no | The submitted YouTube URL. |
| | `text` | `String` | no | Full transcript text. May be empty (J6). |
| | `durationSeconds` | `int` | no | Video duration in seconds. |
| | `chapters` | `List<Chapter>` | no | Possibly empty when no chapter markers exist. |
| | `extractedAt` | `Instant` | no | When the TRANSCRIPT task returned. |
| `KeyPoint` | `pointId` | `String` | no | Short stable id (`p-<8 hex>`). |
| | `text` | `String` | no | The key point, paraphrased from the transcript. |
| | `sourceStartSeconds` | `int` | no | Maps the key point back to a chapter `startSeconds`. |
| `SectionOutline` | `sectionId` | `String` | no | Slug. MUST be unique within a `VideoSummary`. |
| | `heading` | `String` | no | Proposed section heading. |
| | `pointIds` | `List<String>` | no | Each entry MUST equal a `KeyPoint.pointId` from the same `VideoSummary`. |
| `VideoSummary` | `keyPoints` | `List<KeyPoint>` | no | Possibly empty. |
| | `sections` | `List<SectionOutline>` | no | Possibly empty. |
| | `summarisedAt` | `Instant` | no | When the SUMMARISE task returned. |
| `SectionBody` | `sectionId` | `String` | no | MUST equal a `SectionOutline.sectionId` from the upstream `VideoSummary`. |
| | `heading` | `String` | no | Section heading (carried forward from `SectionOutline`). |
| | `body` | `String` | no | 3–5 sentences paraphrasing the section's key points. |
| `BlogDraft` | `introduction` | `String` | no | 2–3 sentence introduction. |
| | `sections` | `List<SectionBody>` | no | Possibly empty. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `SourceTimestamp` | `label` | `String` | no | Display label (typically the chapter title). |
| | `startSeconds` | `int` | no | MUST equal a `Chapter.startSeconds` from the upstream `Transcript`. |
| `PostSection` | `sectionId` | `String` | no | MUST equal a `SectionBody.sectionId` from the upstream `BlogDraft`. |
| | `heading` | `String` | no | Polished section heading. Non-empty (E1 rule 2). |
| | `body` | `String` | no | Polished 3–5 sentence body. Non-empty (E1 rule 4). |
| | `sourceTimestamps` | `List<SourceTimestamp>` | no | Possibly empty; each timestamp traces to the `Transcript`. |
| `BlogPost` | `title` | `String` | no | 1-line title. |
| | `introduction` | `String` | no | Polished 2–3 sentence introduction. |
| | `sections` | `List<PostSection>` | no | `sections.size() == draft.sections.size()` (E1 rule 1). |
| | `conclusion` | `String` | no | 2–3 sentence conclusion. |
| | `wordCount` | `int` | no | Actual word count of `introduction + bodies + conclusion`. 400–1200 target (E1 rule 3). |
| | `polishedAt` | `Instant` | no | When the POLISH task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks satisfied"). |
| | `evaluatedAt` | `Instant` | no | When `EditorialScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `POLISH` (only the POLISH task's response is screened). |
| | `reason` | `String` | no | Structured reason from `PublishGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `BlogPostRecord` (entity state) | `postId` | `String` | no | — |
| | `videoUrl` | `Optional<String>` | yes | Populated after `PostCreated`. |
| | `transcript` | `Optional<Transcript>` | yes | Populated after `TranscriptExtracted`. |
| | `summary` | `Optional<VideoSummary>` | yes | Populated after `SummaryProduced`. |
| | `draft` | `Optional<BlogDraft>` | yes | Populated after `DraftWritten`. |
| | `post` | `Optional<BlogPost>` | yes | Populated after `PostPolished`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `BlogPostStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PostCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `BlogPostRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BlogPostStatus`: `CREATED`, `TRANSCRIBING`, `TRANSCRIBED`, `SUMMARISING`, `SUMMARISED`, `DRAFTING`, `DRAFTED`, `POLISHING`, `POLISHED`, `EVALUATED`, `FAILED`.

## Events (`BlogPostEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PostCreated` | `videoUrl: String` | → CREATED |
| `TranscriptStarted` | — | → TRANSCRIBING |
| `TranscriptExtracted` | `transcript: Transcript` | → TRANSCRIBED |
| `SummaryStarted` | — | → SUMMARISING |
| `SummaryProduced` | `summary: VideoSummary` | → SUMMARISED |
| `DraftStarted` | — | → DRAFTING |
| `DraftWritten` | `draft: BlogDraft` | → DRAFTED |
| `PolishStarted` | — | → POLISHING |
| `PostPolished` | `post: BlogPost` | → POLISHED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, reason, rejectedAt` | no status change (audit-only) |
| `PostFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BlogPostRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BlogPostRow` mirrors `BlogPostRecord` exactly. The UI fetches the full row via `GET /api/posts/{id}` and streams updates via `GET /api/posts/sse`.

The view declares ONE query: `getAllPosts: SELECT * AS posts FROM blog_post_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`BlogTasks.java`)

```java
public final class BlogTasks {
  public static final Task<Transcript> EXTRACT_TRANSCRIPT = Task
      .name("Extract transcript")
      .description("Fetch the video transcript and identify chapter markers")
      .resultConformsTo(Transcript.class);

  public static final Task<VideoSummary> SUMMARISE_VIDEO = Task
      .name("Summarise video")
      .description("Extract key points from the transcript and outline blog sections")
      .resultConformsTo(VideoSummary.class);

  public static final Task<BlogDraft> DRAFT_POST = Task
      .name("Draft post")
      .description("Write section bodies from the section outline and key points")
      .resultConformsTo(BlogDraft.class);

  public static final Task<BlogPost> POLISH_POST = Task
      .name("Polish post")
      .description("Refine prose, add conclusion, finalise the blog post")
      .resultConformsTo(BlogPost.class);

  private BlogTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Publish guardrail

`PublishGuardrail` is a `before-agent-response` hook registered on `BlogAgent`. It runs only on the POLISH task's response — after all prose-polishing tool calls have completed, but before `PostPolished` is recorded. The prohibited-phrase list lives in `src/main/resources/config/prohibited-phrases.txt` and is loaded once at startup. Rejection is recorded as a `GuardrailRejected` event on the entity (audit-only; does not transition status). The agent loop retries within its 3-iteration budget on the POLISH task.
