# Data model

Every record, event, enum, and view row the generated system defines.

## `Post` (PostEntity state + PostsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Post id; equals the workflow id and entity id |
| `topic` | `Optional<String>` | yes | The submitted topic |
| `status` | `PostStatus` | no | Lifecycle status |
| `draftedAt` | `Optional<Instant>` | yes | When the draft was recorded |
| `draftTitle` | `Optional<String>` | yes | Drafted headline |
| `draftContent` | `Optional<String>` | yes | Drafted body |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverComment` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `publishedAt` | `Optional<String>` | yes | ISO-8601 publish time |
| `publishedUrl` | `Optional<String>` | yes | Published destination URL |

Every nullable lifecycle field is `Optional<T>` because `Post` is the view row type (Lesson 6). `emptyState()` returns `Post.initial("")` with no `commandContext()` reference (Lesson 3).

## `PostStatus` enum

`DRAFTED`, `APPROVED`, `REJECTED`, `PUBLISHED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `PostDrafted` | `recordDraft` after ContentAgent returns | sets `draftedAt`, `draftTitle`, `draftContent`, status → `DRAFTED` |
| `PostApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverComment`, status → `APPROVED` |
| `PostRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `PostPublished` | `recordPublication` after PublishingAgent returns | sets `publishedAt`, `publishedUrl`, status → `PUBLISHED` |

## Domain records

- `DraftPost(String title, String content)` — ContentAgent result.
- `ApprovalDecision(String approvedBy, String comment)` — approve payload.
- `PublishedPost(String url, String publishedAt)` — PublishingAgent result.

## View row type

`PostsView` rows are `Post`. One query: `getAllPosts` → `SELECT * AS posts FROM posts_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (PublishingTasks.java)

- `DRAFT` — `Task.name(...).resultConformsTo(DraftPost.class)`.
- `PUBLISH` — `Task.name(...).resultConformsTo(PublishedPost.class)`.
