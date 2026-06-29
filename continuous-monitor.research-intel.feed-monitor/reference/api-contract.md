# API contract — feed-monitor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/feed` | — | `200 [ FeedItemState... ]` (sorted newest-first) | `FeedEndpoint` ← `FeedView` |
| `GET` | `/api/feed/{id}` | — | `200 FeedItemState` / `404` | `FeedEndpoint` ← `FeedView` |
| `POST` | `/api/feed/{id}/override` | `{ "decidedBy": String, "targetChannel": String }` | `200 FeedItemState` (status → NOTIFY_REQUESTED) | `FeedEndpoint` → `FeedItemEntity` |
| `POST` | `/api/feed/{id}/suppress` | `{ "decidedBy": String }` | `200 FeedItemState` (status → SUPPRESSED) | `FeedEndpoint` → `FeedItemEntity` |
| `GET` | `/api/feed/sse` | — | `text/event-stream` | `FeedEndpoint` ← `FeedView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### FeedItemState

```json
{
  "itemId": "fi-3ae7c1",
  "raw": {
    "itemId": "fi-3ae7c1",
    "feedUrl": "https://example.com/feed.rss",
    "title": "EU AI Act enforcement timeline updated",
    "rawContent": "The European Commission has published...",
    "link": "https://example.com/articles/eu-ai-act-timeline",
    "fetchedAt": "2026-06-28T08:00:00Z"
  },
  "summary": {
    "summary": "The European Commission updated the AI Act enforcement timeline, shifting the general-purpose AI obligations to Q3 2026. National competent authorities are expected to begin issuing guidance by August.",
    "topics": ["eu-ai-act", "compliance", "enforcement"],
    "keyQuote": "National competent authorities are expected to begin issuing guidance by August.",
    "summarizedAt": "2026-06-28T08:00:04Z"
  },
  "classification": {
    "classification": "NOTIFY",
    "confidence": "high",
    "reason": "Material regulatory update affecting enterprise AI compliance timelines.",
    "targetChannel": "#research-intel"
  },
  "post": {
    "channel": "#research-intel",
    "messageText": "EU AI Act enforcement timeline updated...",
    "postedAt": "2026-06-28T08:00:07Z",
    "guardrailPassed": true
  },
  "evalScore": 5,
  "evalRationale": "Summary is accurate, specific, and directly relevant to the channel's stated purpose.",
  "status": "POSTED",
  "createdAt": "2026-06-28T08:00:00Z",
  "finishedAt": "2026-06-28T08:00:07Z"
}
```

### SSE event format

```
event: feed-update
data: { "itemId": "fi-3ae7c1", "status": "POSTED", ... }
```

One event per state transition. Clients reconcile by `itemId`.

### Override request body

```json
{ "decidedBy": "analyst-07", "targetChannel": "#research-intel" }
```

### Suppress request body

```json
{ "decidedBy": "analyst-07" }
```
