# API contract — inbox-watcher

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/inbox` | — | `200 [ InboxMessage... ]` (sorted newest-first) | `InboxEndpoint` ← `InboxView` |
| `GET` | `/api/inbox/{id}` | — | `200 InboxMessage` / `404` | `InboxEndpoint` ← `InboxView` |
| `POST` | `/api/inbox/{id}/approve` | `{ "decidedBy": String }` | `200 InboxMessage` (status now SENT) | `InboxEndpoint` → `InboxMessageEntity` |
| `POST` | `/api/inbox/{id}/reject` | `{ "decidedBy": String, "reason": String }` | `200 InboxMessage` (status now DISMISSED) | `InboxEndpoint` → `InboxMessageEntity` |
| `GET` | `/api/inbox/sse` | — | `text/event-stream` | `InboxEndpoint` ← `InboxView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboxMessage

```json
{
  "messageId": "mq-2bf…",
  "incoming": { "messageId": "mq-2bf…", "from": "[REDACTED-EMAIL]", "subject": "Order #[REDACTED-ORDER-ID] hasn't shipped",
                "body": "...", "receivedAt": "2026-06-28T07:45:00Z" },
  "sanitized": { "redactedSubject": "Order #[REDACTED] hasn't shipped", "redactedBody": "...",
                 "piiCategoriesFound": ["email", "order-id"] },
  "classification": { "kind": "AUTO_DRAFTED", "confidence": "high", "reason": "Routine order-status enquiry." },
  "draft": { "subject": "Re: Order #[REDACTED] hasn't shipped", "body": "...", "draftedAt": "2026-06-28T07:45:08Z" },
  "decision": { "approved": true, "decidedBy": "agent-19", "reason": null, "decidedAt": "2026-06-28T07:46:21Z" },
  "evalScore": 4,
  "evalRationale": "Reply is warm, factually grounded, no policy issues.",
  "status": "SENT",
  "createdAt": "2026-06-28T07:45:00Z",
  "finishedAt": "2026-06-28T07:46:21Z"
}
```

### SSE event format

```
event: inbox-update
data: { "messageId": "...", "status": "AWAITING_APPROVAL", ... }
```

One event per state transition. Clients reconcile by `messageId`.
