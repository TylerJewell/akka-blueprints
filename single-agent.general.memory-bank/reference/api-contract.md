# API contract — memory-bank

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/memories` | `SubmitMemoryRequest` | `201 { memoryId }` | `MemoryEndpoint` → `MemoryEntity` |
| `POST` | `/api/memories/recall` | `SubmitMemoryRequest` (operation=RECALL) | `201 { memoryId }` | `MemoryEndpoint` → `MemoryEntity` |
| `GET` | `/api/memories` | — | `200 [ Memory... ]` (newest-first) | `MemoryEndpoint` ← `MemoryView` |
| `GET` | `/api/memories/{id}` | — | `200 Memory` / `404` | `MemoryEndpoint` ← `MemoryView` |
| `GET` | `/api/memories/sse` | — | `text/event-stream` | `MemoryEndpoint` ← `MemoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitMemoryRequest (request body)

```json
{
  "operation": "REMEMBER",
  "namespace": "PERSONAL_ASSISTANT",
  "rawContent": "My preferred meeting time is 9 AM. Contact me at jane.smith@example.com.",
  "submittedBy": "user-42"
}
```

For RECALL operations:

```json
{
  "operation": "RECALL",
  "namespace": "PERSONAL_ASSISTANT",
  "rawContent": "What time do I prefer for meetings?",
  "submittedBy": "user-42"
}
```

### Memory (response body — REMEMBER, terminal state)

```json
{
  "memoryId": "m-7ac...",
  "request": {
    "memoryId": "m-7ac...",
    "operation": "REMEMBER",
    "namespace": "PERSONAL_ASSISTANT",
    "rawContent": "(raw text preserved for audit)",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "sanitized": {
    "redactedContent": "My preferred meeting time is 9 AM. Contact me at [REDACTED-EMAIL].",
    "piiCategoriesFound": ["email"]
  },
  "confirmation": {
    "memoryId": "m-7ac...",
    "acknowledgement": "Stored: preferred meeting time of 9 AM in personal-assistant namespace.",
    "tags": ["meeting-time", "morning-preference", "9am"],
    "confirmedAt": "2026-06-28T09:00:08Z"
  },
  "recallResult": null,
  "status": "STORED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:08Z"
}
```

### Memory (response body — RECALL, terminal state)

```json
{
  "memoryId": "m-b3d...",
  "request": {
    "memoryId": "m-b3d...",
    "operation": "RECALL",
    "namespace": "PERSONAL_ASSISTANT",
    "rawContent": "What time do I prefer for meetings?",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T09:05:00Z"
  },
  "sanitized": {
    "redactedContent": "What time do I prefer for meetings?",
    "piiCategoriesFound": []
  },
  "confirmation": null,
  "recallResult": {
    "queryContent": "What time do I prefer for meetings?",
    "matches": [
      {
        "memoryId": "m-7ac...",
        "sanitizedContent": "My preferred meeting time is 9 AM. Contact me at [REDACTED-EMAIL].",
        "tags": ["meeting-time", "morning-preference", "9am"],
        "relevanceScore": 0.92,
        "storedAt": "2026-06-28T09:00:08Z"
      }
    ],
    "recalledAt": "2026-06-28T09:05:12Z"
  },
  "status": "STORED",
  "createdAt": "2026-06-28T09:05:00Z",
  "finishedAt": "2026-06-28T09:05:12Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: memory-update
data: { "memoryId": "m-7ac...", "status": "STORED", "confirmation": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `STORING`, `STORED`, `FAILED`). Clients reconcile by `memoryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `submittedBy` from the authenticated principal rather than the request body. Memory namespaces should be scoped per authenticated user in a production deployment.
