# Data model — wp-autotagger

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaggingConfig` | `maxTags` | `int` | no | Upper bound on the tag list (1–15). |
| | `tagStyle` | `TagStyle` | no | Enum value; controls the agent's tagging approach. |
| | `submittedBy` | `String` | no | User identifier supplied at submission. |
| `PostRequest` | `jobId` | `String` | no | UUID minted by `PostEndpoint`. |
| | `postUrl` | `String` | no | WordPress post URL to tag. |
| | `config` | `TaggingConfig` | no | Tagging parameters. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `PostBody` | `postId` | `String` | no | WordPress post ID resolved from the URL. |
| | `title` | `String` | no | Post title from the stub/API. |
| | `body` | `String` | no | Full post body text (may contain redacted tokens). |
| | `author` | `String` | no | Author name from the stub/API. |
| | `publishedAt` | `Instant` | no | Original publication timestamp. |
| `TagEntry` | `tag` | `String` | no | Lowercase tag string; alphanumeric, hyphens, spaces. |
| | `confidence` | `double` | no | 0.0–1.0; agent's certainty. |
| `TagProposal` | `tags` | `List<TagEntry>` | no | Ordered list, highest confidence first. |
| | `tagStyle` | `TagStyle` | no | Style used by the agent. |
| | `rationale` | `String` | no | 1–2 sentences from the agent. |
| | `proposedAt` | `Instant` | no | When the agent returned. |
| `Post` (entity state) | `jobId` | `String` | no | — |
| | `request` | `Optional<PostRequest>` | yes | Populated after `PostIngested`. |
| | `body` | `Optional<PostBody>` | yes | Populated after `PostBodyFetched`. |
| | `proposal` | `Optional<TagProposal>` | yes | Populated after `TagsApplied`. |
| | `status` | `PostStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PostIngested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Post` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TagStyle`: `DESCRIPTIVE`, `KEYWORD_DENSE`, `MIXED`.
`PostStatus`: `INGESTED`, `BODY_FETCHED`, `TAGGING`, `TAGS_APPLIED`, `FAILED`.

## Events (`PostEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PostIngested` | `request` | → INGESTED |
| `PostBodyFetched` | `body` | → BODY_FETCHED |
| `TaggingStarted` | — | → TAGGING |
| `TagsApplied` | `proposal` | → TAGS_APPLIED (terminal happy) |
| `PostFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Post.initial("")` with all `Optional` fields as `Optional.empty()` and `status = INGESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PostRow` mirrors `Post` minus `body.body` beyond the first 500 characters (the audit log keeps the full body; the view stores a `bodyPreview: String` field capped at 500 chars for the UI). The UI fetches the full body on demand via `GET /api/posts/{id}` and reads `body.body` from the JSON.

The view declares ONE query: `getAllPosts: SELECT * AS posts FROM post_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`TaggingTasks.java`)

```java
public final class TaggingTasks {
  public static final Task<TagProposal> PROPOSE_TAGS = Task
      .name("Propose tags")
      .description("Read the attached post body and produce a TagProposal with SEO-relevant tags")
      .resultConformsTo(TagProposal.class);

  private TaggingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
