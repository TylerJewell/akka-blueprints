# API contract — ig-dm-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/messages` | `SubmitMessageRequest` | `201 { messageId }` | `DmEndpoint` → `DmEntity` |
| `GET` | `/api/messages` | — | `200 [ DmMessage... ]` (newest-first) | `DmEndpoint` ← `DmView` |
| `GET` | `/api/messages/{id}` | — | `200 DmMessage` / `404` | `DmEndpoint` ← `DmView` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `DmEndpoint` ← `DmView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitMessageRequest (request body)

```json
{
  "senderId": "ig_user_48291",
  "rawText": "Hey! Is the limited edition hoodie still available in XL? Also, can you ship to jane.doe@example.com?",
  "profileId": "retail-brand-v1"
}
```

### DmMessage (response body)

```json
{
  "messageId": "dm-7f3a...",
  "inbound": {
    "messageId": "dm-7f3a...",
    "senderId": "ig_user_48291",
    "rawText": "Hey! Is the limited edition hoodie still available in XL? Also, can you ship to jane.doe@example.com?",
    "profileId": "retail-brand-v1",
    "receivedAt": "2026-06-28T10:15:00Z"
  },
  "sanitized": {
    "sanitizedText": "Hey! Is the limited edition hoodie still available in XL? Also, can you ship to [REDACTED-EMAIL]?",
    "piiCategoriesFound": ["email"]
  },
  "reply": {
    "replyText": "Hi! The limited edition hoodie in XL is available right now. You can place your order on our website and choose your preferred shipping option at checkout. Let us know if you need any help!",
    "toneLabel": "FRIENDLY",
    "toneConfidenceScore": 4,
    "decidedAt": "2026-06-28T10:15:22Z"
  },
  "status": "REPLIED",
  "createdAt": "2026-06-28T10:15:00Z",
  "finishedAt": "2026-06-28T10:15:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: message-update
data: { "messageId": "dm-7f3a...", "status": "REPLIED", "reply": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `REPLYING`, `REPLIED`, `FAILED`). Clients reconcile by `messageId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `senderId` from the authenticated social-channel principal rather than the request body.
