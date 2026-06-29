# API contract — editorial-desk

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/stories` | `{ "topic": String, "requestedBy"?: String }` | `202 { "storyId": String }` | `EditorialEndpoint` → `StoryQueue` |
| `GET` | `/api/stories` | — | `200 [ Document... ]` | `EditorialEndpoint` ← `DocumentBoardView` |
| `GET` | `/api/stories?status=…` | — | `200 [ Document... ]` (filtered client-side) | `EditorialEndpoint` ← `DocumentBoardView` |
| `GET` | `/api/stories/{id}` | — | `200 Document` / `404` | `EditorialEndpoint` ← `DocumentEntity` |
| `GET` | `/api/stories/sse` | — | `text/event-stream` (one event per document change) | `EditorialEndpoint` ← `DocumentBoardView` |
| `GET` | `/api/stories/{id}/sections` | — | `200 [ Section... ]` | `EditorialEndpoint` ← `SectionBoardView` |
| `GET` | `/api/sections/sse` | — | `text/event-stream` (one event per section change) | `EditorialEndpoint` ← `SectionBoardView` |
| `POST` | `/api/stories/{id}/compliance-review` | `{ "reviewedBy": String, "outcome": "CLEARED"\|"FLAGGED", "comments": String }` | `200 Document` / `409` (story not `PUBLISHED`) / `404` | `EditorialEndpoint` → `DocumentEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Document

```json
{
  "storyId": "s-7c1…",
  "topic": "How tides work",
  "requestedBy": "ui",
  "status": "PUBLISHED",
  "brief": {
    "angle": "Explain tides as the everyday result of the moon's and sun's pull.",
    "keyQuestions": ["What causes the two daily high tides?", "How do spring and neap tides differ?"],
    "targetSections": ["What a tide is", "The moon's pull", "Spring and neap tides", "Why coastlines differ"]
  },
  "researchDigest": {
    "summary": "Tides are the rotation of the earth under two gravitational bulges raised by the moon and sun.",
    "keyFacts": ["The moon raises two bulges", "Most coasts see two high tides a day", "Spring tides align sun and moon"]
  },
  "sectionIds": ["s-7c1-s0", "s-7c1-s1", "s-7c1-s2", "s-7c1-s3"],
  "draft": { "headline": "How the tides work", "wordCount": 612 },
  "reviewVerdict": { "outcome": "PASS", "mustRevise": [] },
  "article": { "headline": "How the Tides Work", "byline": "Editorial Desk" },
  "publishedUrl": "https://desk.example/s-7c1",
  "publishedAt": "2026-06-28T08:14:02Z",
  "stageEvals": [
    { "stage": "research", "score": 88, "flags": [], "evaluatedAt": "2026-06-28T08:12:40Z" },
    { "stage": "draft", "score": 81, "flags": [], "evaluatedAt": "2026-06-28T08:13:30Z" },
    { "stage": "review", "score": 90, "flags": [], "evaluatedAt": "2026-06-28T08:13:58Z" }
  ],
  "complianceReview": null,
  "revisionCount": 0,
  "createdAt": "2026-06-28T08:11:50Z"
}
```

The `Document` returned by `GET /api/stories/{id}` carries the full `article.body` and `draft.sections`; the board rows streamed over SSE drop those heavy fields (see `data-model.md`). Lifecycle fields (`brief`, `researchDigest`, `draft`, `reviewVerdict`, `article`, `publishedUrl`, `publishedAt`, `complianceReview`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### Section

```json
{
  "sectionId": "s-7c1-s1",
  "storyId": "s-7c1",
  "title": "The moon's pull",
  "brief": "Explain the gravitational cause and the two bulges.",
  "status": "WRITTEN",
  "claimedBy": "writer-2",
  "claimedAt": "2026-06-28T08:13:01Z",
  "wordCount": 154,
  "writtenAt": "2026-06-28T08:13:22Z",
  "createdAt": "2026-06-28T08:12:55Z"
}
```

The board row carries `wordCount` and a `contentPresent` flag rather than the full `content`; `GET /api/stories/{id}/sections` may include the content for the App UI's per-section expand.

### ComplianceReview (request and recorded form)

```json
{ "reviewedBy": "compliance-1", "outcome": "FLAGGED", "comments": "Headline overstates certainty; recommend a hedge." }
```

Recorded on the document as `complianceReview` with an added `reviewedAt`. Accepted only when the document status is `PUBLISHED`; otherwise the endpoint returns `409`.

### SSE event format

```
event: story-update
data: { "storyId": "s-7c1", "status": "REVIEWING", ... }

event: section-update
data: { "sectionId": "s-7c1-s1", "status": "WRITTEN", ... }
```

`GET /api/stories/sse` emits one `story-update` per document state transition; `GET /api/sections/sse` emits one `section-update` per section transition. Clients reconcile stories by `storyId` and group the section board by `status`.
