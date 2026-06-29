# Data model

Authoritative. `/akka:implement` writes these records exactly. Nullable lifecycle fields are `Optional<T>` (Lesson 6); Akka's Jackson config serializes `Optional<T>` as the raw value or `null`.

## PostContent (PostEntity state + PostsView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Post id (= workflow id) |
| `brief` | `String` | no | The creative brief submitted |
| `status` | `PostStatus` | no | Lifecycle state |
| `composedAt` | `Optional<Instant>` | yes | When the caption was written |
| `caption` | `Optional<String>` | yes | Instagram caption |
| `hashtags` | `List<String>` | no (empty until set) | Hashtags; empty list before compose, never null |
| `promptedAt` | `Optional<Instant>` | yes | When the image prompt was written |
| `imagePrompt` | `Optional<String>` | yes | Image-generation prompt |
| `safetyCheckPassed` | `Optional<Boolean>` | yes | Brand/platform check result |
| `safetyCheckNotes` | `Optional<String>` | yes | Why the check passed or failed |
| `readyAt` | `Optional<Instant>` | yes | When the post was marked ready |
| `blockedAt` | `Optional<Instant>` | yes | When the post was blocked |
| `blockReason` | `Optional<String>` | yes | Why the post was blocked |

`emptyState()` returns `PostContent.initial("", "")` with placeholder identity — no `commandContext()` reference (Lesson 3).

## PostStatus enum

`COMPOSING`, `PROMPTING`, `READY`, `BLOCKED`, `FAILED`.

## Events (PostEntity)

| Event | Trigger |
|---|---|
| `CaptionWritten` | caption step records the `CaptionDraft` after a passing check |
| `ImagePromptWritten` | image-prompt step records the `ImagePrompt` after a passing check |
| `PostReady` | finalize step marks the post ready |
| `PostBlocked` | a failed before-agent-response check at either step |

## BriefQueue

State: a list of queued briefs. Event: `BriefQueued` (trigger: `enqueue(brief)` from `PostEndpoint` or `BriefSimulator`). `BriefConsumer` subscribes and starts a `PostWorkflow` per event.

## Typed agent results

| Record | Fields |
|---|---|
| `CaptionDraft` | `String caption`, `List<String> hashtags` |
| `ImagePrompt` | `String prompt` |
| `ImagePromptInput` | `String brief`, `String caption` |

## View row

`PostsView` row type is `PostContent`. One query: `getAllPosts` → `SELECT * AS posts FROM posts_view`. No `WHERE status` filter (Akka cannot auto-index enum columns, Lesson 2); status filtering happens client-side in the endpoint.
