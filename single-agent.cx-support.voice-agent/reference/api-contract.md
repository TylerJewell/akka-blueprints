# API contract — voice-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `StartConversationRequest` | `201 { conversationId, turnId }` | `ConversationEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{id}/turns` | `AddTurnRequest` | `201 { turnId }` | `ConversationEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ ConversationSession... ]` (newest-first) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 ConversationSession` / `404` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartConversationRequest (request body — POST /api/conversations)

```json
{
  "callerId": "caller-84712",
  "rawTranscript": "Hi, I was charged twice on my account last month and I'd like to get that sorted out.",
  "audioFormat": "simulated"
}
```

### AddTurnRequest (request body — POST /api/conversations/{id}/turns)

```json
{
  "rawTranscript": "It was in May, around the 15th. The charge was for £29.99.",
  "audioFormat": "simulated"
}
```

### ConversationSession (response body)

```json
{
  "conversationId": "c-4f7a...",
  "callerId": "caller-84712",
  "sessionStatus": "ACTIVE",
  "startedAt": "2026-06-28T14:20:00Z",
  "closedAt": null,
  "turns": [
    {
      "turnId": "t-a3c1...",
      "request": {
        "conversationId": "c-4f7a...",
        "turnId": "t-a3c1...",
        "callerId": "caller-84712",
        "rawTranscript": "(raw text preserved for audit)",
        "audioFormat": "simulated",
        "receivedAt": "2026-06-28T14:20:00Z"
      },
      "sanitized": {
        "redactedTranscript": "Hi, I was charged twice on my account last month and I'd like to get that sorted out.",
        "piiCategoriesFound": []
      },
      "reply": {
        "replyText": "I'm sorry to hear about the duplicate charge. Let me look into that for you — could you confirm the billing period so I can find the right transaction?",
        "tone": "WARM",
        "topicClassification": "billing",
        "escalationFlag": false,
        "repliedAt": "2026-06-28T14:20:14Z"
      },
      "audio": {
        "audioBytes": "(base64-encoded WAV bytes)",
        "audioFormat": "wav",
        "durationMs": 4200,
        "synthesisedAt": "2026-06-28T14:20:15Z"
      },
      "status": "AUDIO_SYNTHESISED",
      "createdAt": "2026-06-28T14:20:00Z",
      "finishedAt": "2026-06-28T14:20:15Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-4f7a...", "turnId": "t-a3c1...", "status": "REPLY_RECORDED", "reply": { ... }, ... }
```

One event per state transition (`AUDIO_RECEIVED`, `TRANSCRIPT_SANITIZED`, `RESPONDING`, `REPLY_RECORDED`, `AUDIO_SYNTHESISED`, `FAILED`). Clients reconcile by `conversationId` + `turnId`; an event always carries the full turn at the moment of transition, so a late-joining client never needs to replay earlier events.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `callerId` from the authenticated telephony session rather than the request body.
