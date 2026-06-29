# API contract — akka-stateful-memory-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `CreateConversationRequest` | `201 { conversationId }` | `ConversationEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{id}/turns` | `SubmitTurnRequest` | `202 { turnId }` | `ConversationEndpoint` → `ConversationEntity` + `ConversationWorkflow` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` (newest-first) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}/sse` | — | `text/event-stream` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateConversationRequest (request body)

```json
{
  "initialPersona": "You are a helpful assistant named Memo. You speak concisely and remember what users tell you."
}
```

`initialPersona` is optional. If omitted, the persona block is bootstrapped by the agent on the first turn.

### SubmitTurnRequest (request body)

```json
{
  "userMessage": "I work in distributed systems and mostly write Go.",
  "submittedBy": "user-77"
}
```

### Conversation (response body — full)

```json
{
  "conversationId": "c-3ae...",
  "personaBlock": {
    "blockId": "persona",
    "content": "You are a helpful assistant named Memo. You speak concisely and remember what users tell you.",
    "lastUpdatedAt": "2026-06-28T09:00:00Z"
  },
  "humanBlock": {
    "blockId": "human",
    "content": "- Works in distributed systems\n- Prefers Go over Python\n- Timezone: UTC-5",
    "lastUpdatedAt": "2026-06-28T09:02:18Z"
  },
  "history": [
    { "role": "user",  "text": "I work in distributed systems and mostly write Go.", "sentAt": "2026-06-28T09:00:12Z" },
    { "role": "agent", "text": "Got it — I will keep examples in Go. What are you working on?", "sentAt": "2026-06-28T09:00:30Z" }
  ],
  "latestDrift": null,
  "status": "ACTIVE",
  "createdAt": "2026-06-28T09:00:00Z",
  "lastActivityAt": "2026-06-28T09:00:30Z"
}
```

`latestDrift` is `null` until the first drift evaluation fires (turn 10). After that it holds the most recent `DriftAnnotation`.

Any lifecycle field that has not been populated yet serialises as JSON `null` (Akka serialises `Optional.empty()` as `null`, not as `{}` or `{ "value": null }`).

### ConversationRow (list response)

The list endpoint returns `ConversationRow` objects. A `ConversationRow` mirrors `Conversation` with `history` truncated to the last 50 messages. The full history is available on the single-conversation endpoint.

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-3ae...", "status": "ACTIVE", "humanBlock": { ... }, "latestDrift": null, ... }
```

One event is emitted per entity event: `ConversationCreated`, `UserMessageReceived`, `AgentResponseReady`, `MemoryPatchApplied`, `DriftEvaluated`, `ConversationArchived`. The event payload is the full `ConversationRow` at the moment of the transition, so a late-joining client never needs to replay. Clients reconcile by `conversationId`.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `submittedBy` from the authenticated principal rather than the request body.
