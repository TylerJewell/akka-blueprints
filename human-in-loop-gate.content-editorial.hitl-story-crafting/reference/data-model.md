# Data model

Every record, event, enum, and view row the generated system defines.

## `Story` (StoryEntity state + StoriesView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Story id; equals the workflow id and entity id |
| `premise` | `Optional<String>` | yes | The submitted story premise |
| `status` | `StoryStatus` | no | Lifecycle status |
| `turnCount` | `int` | no | Number of chapters written so far |
| `chapters` | `List<Chapter>` | no | Ordered list of chapters; empty until first draft |
| `startedAt` | `Optional<Instant>` | yes | When the story was created |
| `lastChapterAt` | `Optional<Instant>` | yes | When the most recent chapter was recorded |
| `completedAt` | `Optional<Instant>` | yes | When the reader ended the story |
| `screenedOutAt` | `Optional<Instant>` | yes | When the content guard terminated the story |
| `screenOutReason` | `Optional<String>` | yes | The content guard's rejection reason |

Every nullable field is `Optional<T>` because `Story` is the view row type (Lesson 6). `emptyState()` returns `Story.initial("")` with no `commandContext()` reference (Lesson 3).

## `Chapter` (embedded record, carried inside `Story`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `turnNumber` | `int` | no | 1-based chapter index |
| `title` | `String` | no | Chapter heading from StoryDrafterAgent |
| `body` | `String` | no | Chapter body from StoryDrafterAgent |
| `direction` | `Optional<String>` | yes | Reader direction that led to this chapter; absent on chapter 1 |

## `StoryStatus` enum

`AWAITING_DIRECTION`, `DRAFTING`, `SCREENED_OUT`, `COMPLETED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `StoryStarted` | `startStory` command | sets `startedAt`, `premise`, status → `DRAFTING` |
| `ChapterAdded` | `chapterAdded` after ContentGuardAgent passes | appends chapter to `chapters`, increments `turnCount`, sets `lastChapterAt`, status → `AWAITING_DIRECTION` |
| `DirectionProvided` | `directionProvided` command (reader via API) | stores `direction` on current chapter, status → `DRAFTING` |
| `StoryCompleted` | `storyCompleted` command (reader via API) | sets `completedAt`, status → `COMPLETED` |
| `StoryScreenedOut` | `storyScreenedOut` command (workflow on guard reject) | sets `screenedOutAt`, `screenOutReason`, status → `SCREENED_OUT` |

## Domain records

- `ChapterDraft(String title, String body)` — StoryDrafterAgent result.
- `ReaderDirection(String direction)` — continue payload.
- `GuardResult(boolean passed, String reason)` — ContentGuardAgent result.

## View row type

`StoriesView` rows are `Story`. One query: `getAllStories` → `SELECT * AS stories FROM stories_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (StoryTasks.java)

- `DRAFT_CHAPTER` — `Task.name(...).resultConformsTo(ChapterDraft.class)`.
- `SCREEN_DRAFT` — `Task.name(...).resultConformsTo(GuardResult.class)`.
