# API contract — managed-harness-tool-use-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` + `QueryWorkflow` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "question": "Why is the checkout service p99 latency above 2 s?",
  "profileId": "ops-read-only",
  "requestedBy": "sre-engineer-04"
}
```

### Query (response body)

```json
{
  "queryId": "q-7a3...",
  "request": {
    "queryId": "q-7a3...",
    "question": "Why is the checkout service p99 latency above 2 s?",
    "profileId": "ops-read-only",
    "requestedBy": "sre-engineer-04",
    "submittedAt": "2026-06-29T09:10:00Z"
  },
  "answer": {
    "prose": "Checkout p99 latency reached 3.4 s at 14:22 UTC, correlating with 42 ERROR log lines pointing to a slow query in OrderRepository. Review the database query plan for the order lookup path.",
    "toolTrace": [
      {
        "callId": "tc-001",
        "toolName": "query_metrics",
        "arguments": { "namespace": "checkout", "metric": "http_request_duration_p99", "window": "1h" },
        "outputSummary": "p99 latency peaked at 3.4 s at 14:22 UTC; baseline is 0.8 s.",
        "blocked": false,
        "invokedAt": "2026-06-29T09:10:04Z"
      },
      {
        "callId": "tc-002",
        "toolName": "fetch_logs",
        "arguments": { "service": "checkout", "level": "ERROR", "limit": "50" },
        "outputSummary": "42 ERROR lines in the same window, all referencing a slow database query in OrderRepository.",
        "blocked": false,
        "invokedAt": "2026-06-29T09:10:06Z"
      }
    ],
    "toolCallCount": 2,
    "blockedCallCount": 0,
    "answeredAt": "2026-06-29T09:10:08Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-29T09:10:00Z",
  "finishedAt": "2026-06-29T09:10:08Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7a3...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `ANSWERED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Agent profiles

Three built-in profiles; `profileId` is supplied in the `POST /api/queries` body.

| profileId | Permitted tools |
|---|---|
| `ops-read-only` | `query_metrics`, `list_resources`, `fetch_logs` |
| `ops-read-write` | `query_metrics`, `list_resources`, `fetch_logs`, `compute_stats` |
| `analytics-only` | `query_metrics`, `compute_stats` |

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
