# API contract — multiformat-hybrid-rag

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
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
  "question": "What drove the increase in EU solar capacity additions between 2020 and 2023?",
  "submittedBy": "analyst-07"
}
```

### Query (response body)

```json
{
  "queryId": "q-9a1c...",
  "request": {
    "queryId": "q-9a1c...",
    "question": "What drove the increase in EU solar capacity additions between 2020 and 2023?",
    "submittedBy": "analyst-07",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "retrieval": {
    "chunks": [
      {
        "chunkId": "eu-solar-2023-p4",
        "sourceTitle": "EU Renewable Energy Progress Report 2023",
        "format": "PDF_PAGE",
        "excerpt": "Solar PV additions totalled 56 GW in calendar year 2023...",
        "relevanceScore": 0.87
      },
      {
        "chunkId": "eu-policy-feed-in-md",
        "sourceTitle": "EU Feed-In Tariff Reform Overview",
        "format": "MARKDOWN",
        "excerpt": "Eight member states revised feed-in tariff structures between 2020 and 2022...",
        "relevanceScore": 0.74
      }
    ],
    "totalChunksSearched": 40,
    "retrievedAt": "2026-06-28T14:00:01Z"
  },
  "answer": {
    "decision": "ANSWERED",
    "answerText": "EU solar capacity additions grew from 18 GW in 2020 to 56 GW in 2023. The primary drivers were feed-in tariff reforms across eight member states and falling module prices that reduced the levelised cost of energy for utility-scale projects.",
    "noResultReason": null,
    "citations": [
      {
        "chunkId": "eu-solar-2023-p4",
        "sourceTitle": "EU Renewable Energy Progress Report 2023",
        "format": "PDF_PAGE",
        "passageExcerpt": "Solar PV additions totalled 56 GW in calendar year 2023, the highest annual figure recorded, with Germany, Spain, and Poland accounting for 62 % of new capacity."
      },
      {
        "chunkId": "eu-policy-feed-in-md",
        "sourceTitle": "EU Feed-In Tariff Reform Overview",
        "format": "MARKDOWN",
        "passageExcerpt": "Eight member states revised feed-in tariff structures between 2020 and 2022, reducing administrative barriers and extending contract durations to 20 years."
      }
    ],
    "answeredAt": "2026-06-28T14:00:22Z"
  },
  "status": "ANSWERED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-9a1c...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RETRIEVING`, `ANSWERING`, `ANSWERED`, `NO_RESULT`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
