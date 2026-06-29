# Data model

Authoritative. `/akka:implement` writes these records exactly. Nullable lifecycle fields are `Optional<T>` (Lesson 6); Akka's Jackson config serializes `Optional<T>` as the raw value or `null`.

## PostBrief (PostEntity state + PostsView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Post id (= workflow id) |
| `concept` | `String` | no | The creative concept submitted |
| `status` | `PostStatus` | no | Lifecycle state |
| `researchedAt` | `Optional<Instant>` | yes | When research completed |
| `researchNotes` | `Optional<String>` | yes | Research summary |
| `composedAt` | `Optional<Instant>` | yes | When copy + visual brief completed |
| `caption` | `Optional<String>` | yes | Instagram caption |
| `hashtags` | `List<String>` | no (empty until set) | Hashtags; empty list before compose, never null |
| `visualBrief` | `Optional<String>` | yes | Visual direction |
| `brandCheckPassed` | `Optional<Boolean>` | yes | Brand/platform check result |
| `brandCheckNotes` | `Optional<String>` | yes | Why the check passed or failed |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver |
| `approverComment` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `escalatedAt` | `Optional<Instant>` | yes | When escalated |
| `publishedAt` | `Optional<Instant>` | yes | When published |
| `publishedPermalink` | `Optional<String>` | yes | Generated permalink |

`emptyState()` returns `PostBrief.initial("", "")` with placeholder identity — no `commandContext()` reference (Lesson 3).

## PostStatus enum

`RESEARCHING`, `COMPOSING`, `AWAITING_APPROVAL`, `APPROVED`, `REJECTED`, `ESCALATED`, `PUBLISHED`.

## Events (PostEntity)

| Event | Trigger |
|---|---|
| `PostResearched` | research step records `ResearchNotes` |
| `PostComposed` | compose step records caption + visual brief |
| `BrandChecked` | brand-check step records pass/fail + notes |
| `PostApproved` | `POST /api/posts/{id}/approve` |
| `PostRejected` | `POST /api/posts/{id}/reject` or a failed brand check |
| `PostEscalated` | `StalePostMonitor` marks a stale post |
| `PostPublished` | publish step records the permalink |

## ConceptQueue

State: a list of queued concepts. Event: `ConceptQueued` (trigger: `enqueue(concept)` from `PostEndpoint` or `ConceptSimulator`). `ConceptConsumer` subscribes and starts a `PostWorkflow` per event.

## Typed agent results

| Record | Fields |
|---|---|
| `ResearchNotes` | `String summary`, `List<String> sources` |
| `CaptionDraft` | `String caption`, `List<String> hashtags` |
| `VisualBrief` | `String brief` |
| `ComposeInput` | `String concept`, `String researchSummary` |
| `ApprovalDecision` | `String approvedBy`, `String comment` |
| `RejectDecision` | `String rejectedBy`, `String reason` |

## View row

`PostsView` row type is `PostBrief`. One query: `getAllPosts` → `SELECT * AS posts FROM posts_view`. No `WHERE status` filter (Akka cannot auto-index enum columns, Lesson 2); status filtering happens client-side in the endpoint.
