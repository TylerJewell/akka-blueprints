# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/stories` | `{ "premise": "string" }` | `{ "storyId": "uuid" }` | StoryEndpoint → StoryWorkflow |
| POST | `/api/stories/{storyId}/continue` | `{ "direction": "string" }` | `200` \| `404` \| `409` | StoryEndpoint → StoryEntity |
| POST | `/api/stories/{storyId}/end` | — | `200` \| `404` \| `409` | StoryEndpoint → StoryEntity |
| GET | `/api/stories` | — | `{ "stories": [Story, ...] }` | StoryEndpoint → StoriesView |
| GET | `/api/stories/{storyId}` | — | `Story` \| `404` | StoryEndpoint → StoryEntity |
| GET | `/api/stories/sse` | — | SSE stream of `Story` | StoryEndpoint → StoriesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | StoryEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | StoryEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | StoryEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Response codes

- `409` on `/continue` or `/end` when the story is not in `AWAITING_DIRECTION` (e.g., already `COMPLETED`, `SCREENED_OUT`, or still `DRAFTING`).

## Payload shapes

`Story` (lifecycle and chapter fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "premise": "string or null",
  "status": "AWAITING_DIRECTION | DRAFTING | SCREENED_OUT | COMPLETED",
  "turnCount": 0,
  "startedAt": "ISO-8601 or null",
  "lastChapterAt": "ISO-8601 or null",
  "completedAt": "ISO-8601 or null",
  "screenedOutAt": "ISO-8601 or null",
  "screenOutReason": "string or null",
  "chapters": [
    {
      "turnNumber": 1,
      "title": "string",
      "body": "string",
      "direction": "string or null"
    }
  ]
}
```

## SSE event format

`GET /api/stories/sse` streams the `StoriesView` rows. Each message:

```
event: story
data: { ...Story... }
```

The UI subscribes once and replaces the matching row by `id` on each event. Because `chapters` is embedded on `Story`, each SSE message carries the full chapter list for that story.
