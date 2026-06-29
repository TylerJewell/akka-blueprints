# Data model — akka-llm-pipeline-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `OutlineSection` | `heading` | `String` | no | Section heading; non-blank (enforced by `SchemaGuardrail`). |
| | `keyPoints` | `List<String>` | no | 2-4 bullet strings for the draft writer. |
| `PostOutline` | `title` | `String` | no | Working title; non-blank (enforced by `SchemaGuardrail`). |
| | `wordTarget` | `int` | no | Target word count for the finished draft; > 0 (enforced by `SchemaGuardrail`). |
| | `sections` | `List<OutlineSection>` | no | Non-empty (enforced by `SchemaGuardrail`). |
| | `outlinedAt` | `Instant` | no | When `OutlineActivity` returned. |
| `DraftSection` | `heading` | `String` | no | MUST equal the matching `OutlineSection.heading` exactly (case-preserved; enforced by `EditorialGuardrail`). |
| | `body` | `String` | no | 2-5 sentences of prose. No prohibited phrases (enforced by `EditorialGuardrail`). |
| `BlogPost` | `title` | `String` | no | MUST equal `PostOutline.title`. |
| | `introduction` | `String` | no | 1-2 sentences introducing the post. No prohibited phrases. |
| | `sections` | `List<DraftSection>` | no | `sections.length == outline.sections.length` (enforced by `EditorialGuardrail`). |
| | `conclusion` | `String` | no | 1-2 sentences closing the post. |
| | `wordCount` | `int` | no | Actual word count; within ±20% of `wordTarget` (enforced by `EditorialGuardrail`). |
| | `draftedAt` | `Instant` | no | When `DraftActivity` returned. |
| `EditorialReview` | `passed` | `boolean` | no | `true` → `READY`; `false` → `FLAGGED`. |
| | `reason` | `String` | no | `"all checks passed"` on happy path; structured rule + reason on flag. |
| | `reviewedAt` | `Instant` | no | When `EditorialGuardrail` finished. |
| `ValidationFailure` | `field` | `String` | no | Which field failed: `"title"`, `"sections"`, `"wordTarget"`. |
| | `reason` | `String` | no | Human-readable reason from `SchemaGuardrail`. |
| | `failedAt` | `Instant` | no | When `SchemaGuardrail` fired. |
| `PostRecord` (entity state) | `postId` | `String` | no | — |
| | `topic` | `Optional<String>` | yes | Populated after `PostCreated`. |
| | `wordTarget` | `Optional<Integer>` | yes | Populated after `PostCreated`. |
| | `outline` | `Optional<PostOutline>` | yes | Populated after `OutlineProduced`. |
| | `draft` | `Optional<BlogPost>` | yes | Populated after `DraftProduced`. |
| | `review` | `Optional<EditorialReview>` | yes | Populated after `PostApproved` or `PostFlagged`. |
| | `validationFailure` | `Optional<ValidationFailure>` | yes | Populated after `ValidationFailed`. |
| | `status` | `PostStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PostCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `PostRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PostStatus`: `CREATED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `REVIEWING`, `READY`, `FLAGGED`, `FAILED`.

## Events (`PostEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PostCreated` | `topic: String, wordTarget: int` | → CREATED |
| `OutlineStarted` | — | → OUTLINING |
| `OutlineProduced` | `outline: PostOutline` | → OUTLINED |
| `DraftStarted` | — | → DRAFTING |
| `DraftProduced` | `draft: BlogPost` | → DRAFTED |
| `ReviewStarted` | — | → REVIEWING |
| `PostApproved` | `review: EditorialReview` | → READY (terminal happy) |
| `PostFlagged` | `reason: String, review: EditorialReview` | → FLAGGED (terminal held) |
| `ValidationFailed` | `field: String, reason: String, failedAt: Instant` | → FAILED (terminal) |
| `PostFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PostRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PostRow` mirrors `PostRecord` exactly. The UI fetches the full row via `GET /api/posts/{id}` and streams updates via `GET /api/posts/sse`.

The view declares ONE query: `getAllPosts: SELECT * AS posts FROM post_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Activity definitions

```
OutlineActivity
  input:  (topic: String, wordTarget: int)
  output: PostOutline
  prompt: prompts/outline-activity.md
  guardrail (after-llm-response): SchemaGuardrail

DraftActivity
  input:  PostOutline
  output: BlogPost
  prompt: prompts/draft-activity.md
  guardrail (before-agent-response): EditorialGuardrail
```

There is no `Task<R>` companion class (no `AutonomousAgent`). These are typed LLM activities executed within `ContentWorkflow` steps using the `ModelClient` API.

## SchemaGuardrail rules (H1)

Four checks on `PostOutline`, applied in order:

1. `title` is non-blank.
2. `sections` is non-empty.
3. Every `OutlineSection.heading` is non-blank.
4. `wordTarget > 0`.

First failure triggers `Guardrail.reject(...)` and records `ValidationFailed` on `PostEntity`. All four passing triggers `Guardrail.accept()`.

## EditorialGuardrail rules (H2)

Three checks on `BlogPost`, applied in order:

1. **Word-count constraint**: `abs(blogPost.wordCount - outline.wordTarget) / outline.wordTarget <= 0.20`.
2. **Prohibited-topic check**: none of the phrases in `src/main/resources/editorial-policy/prohibited-phrases.txt` appears (case-insensitive) in `blogPost.introduction` or any `DraftSection.body`.
3. **Section-coverage check**: for every `OutlineSection.heading` in `outline.sections`, there is at least one `DraftSection.heading` in `blogPost.sections` that equals it (case-insensitive).

Any failure triggers `Guardrail.flag(...)` and records `PostFlagged` on `PostEntity`. All three passing triggers `Guardrail.accept()` and records `PostApproved`.
