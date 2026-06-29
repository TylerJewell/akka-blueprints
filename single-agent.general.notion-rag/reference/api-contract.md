# API contract — notion-rag

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `QueryEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{sessionId}/questions` | `SubmitQuestionRequest` | `201 { questionId }` | `QueryEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (newest-first) | `QueryEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{sessionId}` | — | `200 Session` / `404` | `QueryEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{sessionId}/sse` | — | `text/event-stream` | `QueryEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "label": "Product Q&A — sprint 42",
  "createdBy": "analyst-7"
}
```

### SubmitQuestionRequest (request body)

```json
{
  "questionText": "Which products support SSO?",
  "submittedBy": "analyst-7"
}
```

### Session (response body)

```json
{
  "sessionId": "s-9ae...",
  "label": "Product Q&A — sprint 42",
  "questions": [
    {
      "questionId": "q-4bc...",
      "sessionId": "s-9ae...",
      "questionText": "Which products support SSO?",
      "submittedBy": "analyst-7",
      "submittedAt": "2026-06-28T14:00:00Z",
      "retrievedRows": {
        "databaseId": "db-prod-catalogue",
        "queryText": "Which products support SSO?",
        "rows": [
          {
            "rowId": "row-001",
            "properties": {
              "Name": "Acme SSO",
              "SupportsSSO": "true",
              "PricingTier": "Enterprise",
              "Status": "active"
            }
          }
        ],
        "totalRowsFetched": 3
      },
      "answer": {
        "answer": "One product supports SSO: Acme SSO, available on the Enterprise tier.",
        "confidence": "HIGH",
        "citations": [
          {
            "rowId": "row-001",
            "propertyName": "SupportsSSO",
            "excerpt": "true",
            "matchReason": "SupportsSSO is enabled for this product"
          }
        ],
        "answeredAt": "2026-06-28T14:00:18Z"
      },
      "status": "ANSWERED"
    }
  ],
  "status": "ACTIVE",
  "createdAt": "2026-06-28T14:00:00Z",
  "lastActivityAt": "2026-06-28T14:00:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-9ae...", "questionId": "q-4bc...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per question state transition (`SUBMITTED`, `ROWS_RETRIEVED`, `ANSWERING`, `ANSWERED`, `FAILED`). Clients reconcile by `questionId` within the session; an event always carries the full question row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` / `createdBy` from the authenticated principal rather than the request body.
