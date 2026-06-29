# API contract — akka-async-agent-mesh

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/dispatch` | `{ "fromAgent": String, "toAgent": String, "topic": String, "body": String }` | `202 { "requestId": String }` | `MeshEndpoint` → `MessageBusEntity` |
| `GET` | `/api/messages` | — | `200 [ Message... ]` | `MeshEndpoint` ← `MessageBusView` |
| `GET` | `/api/messages?fromAgent=…&toAgent=…&status=…` | — | `200 [ Message... ]` (filtered client-side) | `MeshEndpoint` ← `MessageBusView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `MeshEndpoint` ← `MessageEntity` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` (one event per message status change) | `MeshEndpoint` ← `MessageBusView` |
| `GET` | `/api/agents/{agentId}/memory` | — | `200 AgentMemory` | `MeshEndpoint` ← `AgentMemoryEntity` |
| `GET` | `/api/receipts/{messageId}` | — | `200 DeliveryReceipt` / `404` | `MeshEndpoint` ← `DeliveryReceiptEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String, "by": String }` | `200 SystemControl` | `MeshEndpoint` → `SystemControl` |
| `POST` | `/api/control/resume` | — | `200 SystemControl` | `MeshEndpoint` → `SystemControl` |
| `GET` | `/api/control` | — | `200 SystemControl` | `MeshEndpoint` ← `SystemControl` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Message

```json
{
  "messageId": "msg-7a1…",
  "requestId": "req-3f2…",
  "fromAgent": "agent-alpha",
  "toAgent": "agent-beta",
  "topic": "status-query",
  "body": "What is the current state of the document index?",
  "status": "PROCESSED",
  "deliveryReceiptId": "rcpt-9b3…",
  "deliveredAt": "2026-06-28T09:14:02Z",
  "processingSummary": "Responded to index-status query with last-rebuild date from memory.",
  "blockedReason": null,
  "processedAt": "2026-06-28T09:14:07Z",
  "staledAt": null,
  "createdAt": "2026-06-28T09:14:00Z"
}
```

### AgentMemory

```json
{
  "agentId": "agent-beta",
  "entries": [
    {
      "entryId": "mem-c4d…",
      "agentId": "agent-beta",
      "tag": "index-status",
      "content": "index last rebuilt 2026-06-27",
      "recordedAt": "2026-06-28T09:14:07Z"
    }
  ],
  "lastUpdatedAt": "2026-06-28T09:14:07Z"
}
```

### DeliveryReceipt

```json
{
  "receiptId": "rcpt-9b3…",
  "messageId": "msg-7a1…",
  "fromAgent": "agent-alpha",
  "toAgent": "agent-beta",
  "acknowledgedAt": "2026-06-28T09:14:01Z"
}
```

### SystemControl

```json
{ "halted": false, "haltedReason": null, "haltedBy": null, "haltedAt": null }
```

### SSE event format

```
event: message-update
data: { "messageId": "msg-7a1…", "status": "PROCESSING", ... }
```

One event per `MessageEntity` status transition. Clients reconcile by `messageId` and group the board by `status`.

## Notes on lifecycle fields

`deliveryReceiptId`, `deliveredAt`, `processingSummary`, `blockedReason`, `processedAt`, `staledAt` are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6). A message in `DISPATCHED` state has `null` for all post-dispatch fields. A `SEND_BLOCKED` message has `blockedReason` populated and all other lifecycle fields `null`.
