# API contract — youtube-analyst

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/analyses` | `SubmitAnalysisRequest` | `201 { analysisId }` | `AnalysisEndpoint` → `ChannelEntity` |
| `GET` | `/api/analyses` | — | `200 [ ChannelAnalysis... ]` (newest-first) | `AnalysisEndpoint` ← `AnalysisView` |
| `GET` | `/api/analyses/{id}` | — | `200 ChannelAnalysis` / `404` | `AnalysisEndpoint` ← `AnalysisView` |
| `GET` | `/api/analyses/sse` | — | `text/event-stream` | `AnalysisEndpoint` ← `AnalysisView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAnalysisRequest (request body)

```json
{
  "channelHandle": "@devtoolscreator",
  "period": "28d",
  "focusArea": "ENGAGEMENT",
  "requestedBy": "strategist-42"
}
```

`period` values: `"28d"` | `"90d"` | `"all"`.
`focusArea` values: `"GROWTH"` | `"ENGAGEMENT"` | `"AUDIENCE"` | `"ALL"`.

### ChannelAnalysis (response body)

```json
{
  "analysisId": "a-7c3...",
  "request": {
    "analysisId": "a-7c3...",
    "channelHandle": "@devtoolscreator",
    "period": "28d",
    "focusArea": "ENGAGEMENT",
    "requestedBy": "strategist-42",
    "requestedAt": "2026-06-28T14:00:00Z"
  },
  "metrics": {
    "channelId": "UC_devtools_fixture",
    "channelHandle": "@devtoolscreator",
    "subscriberCount": 84200,
    "subscriberDelta": 4200,
    "totalViews": 1820000,
    "viewsDelta": 187000,
    "period": "28d",
    "topVideos": [
      {
        "videoId": "vid-001",
        "title": "Build a REST API in 10 minutes",
        "views": 82000,
        "likes": 6200,
        "comments": 620,
        "engagementRate": 8.3,
        "publishedAt": "2026-06-01"
      }
    ],
    "fetchedAt": "2026-06-28T14:00:01Z"
  },
  "report": {
    "tier": "GROWING",
    "summary": "The channel added 4,200 subscribers over 28 days, driven by two tutorial videos at 8.3% engagement rate — well above the 5.1% channel average.",
    "topVideos": [
      {
        "videoId": "vid-001",
        "title": "Build a REST API in 10 minutes",
        "views": 82000,
        "engagementRate": 8.3,
        "trend": "UP"
      }
    ],
    "recommendations": [
      {
        "metricRef": "engagementRate",
        "action": "Prioritize short tutorial formats under 15 minutes which consistently outperform longer content on engagement rate."
      },
      {
        "metricRef": "subscriberDelta",
        "action": "Increase upload frequency to weekly during tutorial production periods to sustain current subscriber growth."
      }
    ],
    "reportedAt": "2026-06-28T14:00:22Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Summary cites numeric evidence; all topVideos have non-zero engagement rates; tier classification is consistent with positive subscriberDelta.",
    "evaluatedAt": "2026-06-28T14:00:23Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: analysis-update
data: { "analysisId": "a-7c3...", "status": "REPORT_RECORDED", "report": { ... }, ... }
```

One event per state transition (`REQUESTED`, `METRICS_FETCHED`, `ANALYSING`, `REPORT_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `analysisId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
