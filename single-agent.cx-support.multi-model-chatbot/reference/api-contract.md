# API contract — multi-model-chatbot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/chat/sessions` | `StartSessionRequest` | `201 { sessionId }` | `ChatEndpoint` → `ConversationEntity` |
| `POST` | `/api/chat/{sessionId}/messages` | `SendMessageRequest` | `201 { turnId }` | `ChatEndpoint` → `ConversationEntity` |
| `GET` | `/api/chat/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/chat/{sessionId}` | — | `200 SessionRow` / `404` | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/chat/{sessionId}/sse` | — | `text/event-stream` | `ChatEndpoint` ← `ConversationView` |
| `PUT` | `/api/chat/{sessionId}/provider` | `ChangeProviderRequest` | `204` | `ChatEndpoint` → `ConversationEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "userId": "user-7821",
  "displayName": "Alice T.",
  "activeProvider": "anthropic"
}
```

`activeProvider` is optional; defaults to the value of `akka.javasdk.agent.default` in `application.conf`.

### SendMessageRequest (request body)

```json
{
  "userText": "I was charged twice for my subscription last month. My email is alice@example.com.",
  "requestedProvider": null
}
```

`requestedProvider` is optional; omit to use the session's current `activeProvider`. If supplied, the session's `activeProvider` is updated before the turn is processed.

### ChangeProviderRequest (request body)

```json
{
  "provider": "openai"
}
```

Valid values: `"anthropic"`, `"openai"`, `"googleai-gemini"`. Stored via `ProviderChanged` event; takes effect on the next message turn.

### SessionRow (response body)

```json
{
  "sessionId": "s-9fa...",
  "userId": "user-7821",
  "displayName": "Alice T.",
  "activeProvider": "anthropic",
  "turns": [
    {
      "turnId": "t-4c2...",
      "userText": "I was charged twice for my subscription last month. My email is alice@example.com.",
      "sanitizedText": "I was charged twice for my subscription last month. My email is [REDACTED-EMAIL].",
      "piiCategoriesFound": ["email"],
      "reply": {
        "replyText": "I can see there's a potential duplicate charge question on your account...",
        "providerName": "anthropic",
        "modelId": "claude-sonnet-4-6",
        "inputTokens": 142,
        "outputTokens": 68,
        "generatedAt": "2026-06-28T14:22:01Z"
      },
      "status": "REPLIED",
      "receivedAt": "2026-06-28T14:22:00Z",
      "repliedAt": "2026-06-28T14:22:08Z"
    }
  ],
  "status": "ACTIVE",
  "createdAt": "2026-06-28T14:21:55Z",
  "lastActivityAt": "2026-06-28T14:22:08Z"
}
```

Any lifecycle field on a `MessageTurn` that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-9fa...", "updatedTurnId": "t-4c2...", "turnStatus": "REPLIED", "turns": [ ... ] }
```

One event per turn state transition (`RECEIVED`, `SANITIZED`, `REPLIED`, `FAILED`) and per session event (`ProviderChanged`). The event always carries the full `SessionRow` at the moment of transition, so a late-joining client never needs to replay. Clients reconcile by `sessionId`; `updatedTurnId` identifies which turn changed.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `userId` from the authenticated principal rather than the request body.
