# Data model — http-content-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ContentBrief` | `topic` | `String` | no | What the content covers. Supplied by the caller. |
| | `targetAudience` | `String` | no | Who the content is written for. |
| | `outputFormat` | `OutputFormat` | no | Enum: BLOG_POST, SOCIAL_POST, EMAIL_TEASER, PRODUCT_DESCRIPTION. |
| | `wordCountHint` | `int` | no | Target word count for the body; 50–2000. Default 300 when not supplied. |
| | `submittedBy` | `String` | no | Caller-supplied identity string. |
| | `submittedAt` | `Instant` | no | When `ContentEndpoint` received the request. |
| `GeneratedContent` | `title` | `String` | no | Headline or subject line for the piece. |
| | `body` | `String` | no | Full text of the generated piece. |
| | `format` | `OutputFormat` | no | Must match the `outputFormat` from the brief. |
| | `wordCount` | `int` | no | Actual word count of `body` (split on whitespace). |
| | `generatedAt` | `Instant` | no | When the agent returned the approved draft. |
| `GuardrailResult` | `passed` | `boolean` | no | True when the draft clears all checks. |
| | `failedChecks` | `List<String>` | no | Names of failed checks; empty when passed. |
| `ContentJob` (entity state) | `jobId` | `String` | no | UUID minted by `ContentEndpoint`. |
| | `brief` | `Optional<ContentBrief>` | yes | Populated after `JobRequested`. |
| | `content` | `Optional<GeneratedContent>` | yes | Populated after `ContentApproved`. |
| | `status` | `JobStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `JobRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ContentJob` is `Optional<T>`. The view's table updater wraps values with
`Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`OutputFormat`: `BLOG_POST`, `SOCIAL_POST`, `EMAIL_TEASER`, `PRODUCT_DESCRIPTION`.
`JobStatus`: `REQUESTED`, `GENERATING`, `APPROVED`, `FAILED`.

## Events (`ContentJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobRequested` | `brief` | → REQUESTED |
| `GenerationStarted` | — | → GENERATING |
| `ContentApproved` | `content` | → APPROVED (terminal happy) |
| `JobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ContentJob.initial("")` with all `Optional` fields as `Optional.empty()`
and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ContentRow` mirrors `ContentJob`. The view declares ONE query:
`getAllJobs: SELECT * AS jobs FROM content_job_view`. No `WHERE status = :status` filter — Akka
cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ContentTasks.java`)

```java
public final class ContentTasks {
  public static final Task<GeneratedContent> GENERATE_CONTENT = Task
      .name("Generate content")
      .description("Write content matching the supplied brief and return it as a GeneratedContent record")
      .resultConformsTo(GeneratedContent.class);

  private ContentTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
