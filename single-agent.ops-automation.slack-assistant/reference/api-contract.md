# API contract — slack-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/messages` | `SubmitMessageRequest` | `201 { messageId }` | `MessageEndpoint` → `MessageEntity` |
| `GET` | `/api/messages` | — | `200 [ Message... ]` (newest-first) | `MessageEndpoint` ← `MessageView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `MessageEndpoint` ← `MessageView` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `MessageEndpoint` ← `MessageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitMessageRequest (request body)

```json
{
  "channelId": "C01234ABCDE",
  "channelName": "incidents",
  "triggerUserId": "U0987ZYXWV",
  "triggerText": "Deploy to prod failed at step 3 — logs say 'connection refused'. Who owns this?"
}
```

### Message (response body)

```json
{
  "messageId": "m-7ac...",
  "context": {
    "channelId": "C01234ABCDE",
    "channelName": "incidents",
    "triggerUserId": "U0987ZYXWV",
    "triggerText": "Deploy to prod failed at step 3 — logs say 'connection refused'. Who owns this?",
    "receivedAt": "2026-06-28T14:30:00Z"
  },
  "sanitized": {
    "scrubbedText": "Deploy to prod failed at step 3 — logs say 'connection refused'. Who owns this?",
    "secretCategoriesFound": []
  },
  "reply": {
    "actionType": "ESCALATE",
    "responseText": "Escalating to the on-call SRE. A connection-refused error at deploy step 3 indicates a downstream service is unavailable — check the dependency health dashboard before rerunning.",
    "runbookUrl": null,
    "guardrailIterations": 1,
    "repliedAt": "2026-06-28T14:30:18Z"
  },
  "audit": {
    "candidateCount": 1,
    "rejectedCount": 0,
    "outcome": "accepted-first-iteration",
    "auditedAt": "2026-06-28T14:30:19Z"
  },
  "status": "AUDITED",
  "createdAt": "2026-06-28T14:30:00Z",
  "finishedAt": "2026-06-28T14:30:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: message-update
data: { "messageId": "m-7ac...", "status": "REPLY_RECORDED", "reply": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `REPLYING`, `REPLY_RECORDED`, `AUDITED`, `FAILED`). Clients reconcile by `messageId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding Slack event subscriptions must wrap the endpoint with signature verification (Slack's `X-Slack-Signature` header) and set `triggerUserId` from the verified Slack event payload rather than the request body.
