# API contract — sentiment-monitor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/sentiment/threads` | — | `200 [ IssueThreadRow... ]` (sorted CRITICAL first, then DECLINING, then STABLE) | `SentimentEndpoint` ← `ThreadView` |
| `GET` | `/api/sentiment/threads/{issueId}` | — | `200 IssueThreadRow` with full `scoredComments` list / `404` | `SentimentEndpoint` ← `IssueThreadEntity` |
| `POST` | `/api/sentiment/threads/{issueId}/silence` | `{ "silencedBy": String }` | `200 IssueThreadRow` (status now SILENCED) | `SentimentEndpoint` → `IssueThreadEntity` |
| `POST` | `/api/sentiment/threads/{issueId}/resolve` | `{ "resolvedBy": String }` | `200 IssueThreadRow` (status now RESOLVED) | `SentimentEndpoint` → `IssueThreadEntity` |
| `GET` | `/api/sentiment/sse` | — | `text/event-stream` | `SentimentEndpoint` ← `ThreadView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IssueThreadRow (list view)

```json
{
  "issueId": "ENG-4821",
  "issueTitle": "Payment timeout on checkout",
  "runningAverage": -3.2,
  "trendLabel": "CRITICAL",
  "consecutiveNegativeCount": 6,
  "status": "ACTIVE",
  "firstSeenAt": "2026-06-28T09:00:00Z",
  "criticalAt": "2026-06-28T09:07:30Z",
  "lastAlert": {
    "dispatched": true,
    "channelId": "C_ESCALATIONS",
    "silencedBy": null,
    "decidedAt": "2026-06-28T09:07:35Z"
  },
  "latestScore": { "score": -4, "confidence": "high", "reason": "Customer impact escalating." }
}
```

### IssueThreadRow (detail — full scoredComments)

```json
{
  "issueId": "ENG-4821",
  "issueTitle": "Payment timeout on checkout",
  "runningAverage": -3.2,
  "trendLabel": "CRITICAL",
  "consecutiveNegativeCount": 6,
  "status": "ACTIVE",
  "firstSeenAt": "2026-06-28T09:00:00Z",
  "criticalAt": "2026-06-28T09:07:30Z",
  "lastAlert": { "dispatched": true, "channelId": "C_ESCALATIONS", "silencedBy": null, "decidedAt": "2026-06-28T09:07:35Z" },
  "scoredComments": [
    {
      "commentId": "cmt-001",
      "comment": { "commentId": "cmt-001", "issueId": "ENG-4821", "authorId": "usr-42", "body": "Still broken.", "postedAt": "2026-06-28T09:00:00Z" },
      "score": { "score": -4, "confidence": "high", "reason": "Unresolved complaint with urgency." },
      "scoredAt": "2026-06-28T09:00:03Z"
    }
  ]
}
```

### Silence / Resolve request body

```json
{ "silencedBy": "agent-19" }
```

```json
{ "resolvedBy": "agent-19" }
```

### SSE event format

```
event: thread-update
data: { "issueId": "ENG-4821", "trendLabel": "CRITICAL", "consecutiveNegativeCount": 6, ... }
```

One event per state transition. Clients reconcile by `issueId`.
