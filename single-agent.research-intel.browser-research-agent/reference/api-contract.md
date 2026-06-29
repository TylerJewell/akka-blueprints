# API contract — reddit-search

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `SubmitJobRequest` | `201 { jobId }` | `ResearchEndpoint` → `ResearchJobEntity` |
| `GET` | `/api/jobs` | — | `200 [ ResearchJob... ]` (newest-first) | `ResearchEndpoint` ← `ResearchView` |
| `GET` | `/api/jobs/{id}` | — | `200 ResearchJob` / `404` | `ResearchEndpoint` ← `ResearchView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` | `ResearchEndpoint` ← `ResearchView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitJobRequest (request body)

```json
{
  "topic": "Akka actor model use cases",
  "subredditScope": ["scala", "java", "programming"],
  "maxPages": 10
}
```

`subredditScope` may be an empty array to search across all of Reddit. `maxPages` must be between 1 and 20 inclusive.

### ResearchJob (response body)

```json
{
  "jobId": "j-7f3a...",
  "topic": {
    "topic": "Akka actor model use cases",
    "subredditScope": ["scala", "java", "programming"],
    "maxPages": 10
  },
  "report": {
    "posts": [
      {
        "postId": "t3_abc123",
        "title": "Akka actors saved us from callback hell",
        "subreddit": "r/scala",
        "upvotes": 612,
        "summaryLine": "The author describes migrating a high-throughput pipeline from nested callbacks to Akka actors and seeing a 40% reduction in bug reports.",
        "url": "https://www.reddit.com/r/scala/comments/abc123/akka_actors_saved_us/"
      },
      {
        "postId": "t3_def456",
        "title": "When should you NOT use Akka actors?",
        "subreddit": "r/java",
        "upvotes": 388,
        "summaryLine": "A discussion thread weighing cases where smaller concurrency primitives outperform the overhead of the actor model.",
        "url": "https://www.reddit.com/r/java/comments/def456/when_not_to_use_akka/"
      }
    ],
    "themes": [
      { "phrase": "actor supervision", "occurrences": 5 },
      { "phrase": "message passing", "occurrences": 4 },
      { "phrase": "backpressure handling", "occurrences": 3 }
    ],
    "sentiment": { "positive": 6, "neutral": 3, "negative": 1 },
    "pagesVisited": 8,
    "noResultsReason": null,
    "completedAt": "2026-06-28T14:22:00Z"
  },
  "status": "REPORT_READY",
  "pagesVisited": 8,
  "createdAt": "2026-06-28T14:21:00Z",
  "finishedAt": "2026-06-28T14:22:00Z"
}
```

A job whose report has not yet landed returns `"report": null`. A `BUDGET_EXHAUSTED` job returns a partial report with `pagesVisited` equal to `maxPages`.

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: job-update
data: { "jobId": "j-7f3a...", "status": "BROWSING", "pagesVisited": 3, ... }
```

One event per state transition (`QUEUED`, `BROWSING`, `REPORT_READY`, `BUDGET_EXHAUSTED`, `FAILED`). Clients reconcile by `jobId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay. The `pagesVisited` counter increments on each `PageVisited` event — the SSE stream emits those intermediate events too, allowing the UI to update the live counter.

## Authorization

ACL: open (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set an authenticated `submittedBy` field.
