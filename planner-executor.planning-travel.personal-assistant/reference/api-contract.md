# API contract — line-personal-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/webhook` | LINE Messaging API event payload (signature-verified) | `200 {}` | `LineWebhookEndpoint` → `MessageQueue` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` | `LineWebhookEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `LineWebhookEndpoint` ← `ConversationEntity` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` (one event per conversation change) | `LineWebhookEndpoint` ← `ConversationView` |
| `POST` | `/api/confirmations/{id}/reply` | `{ "approved": boolean }` | `200 { "conversationId": String, "status": String }` | `LineWebhookEndpoint` → `ConfirmationEntity` |
| `GET` | `/api/confirmations/{id}` | — | `200 ConfirmationRequest` / `404` | `LineWebhookEndpoint` ← `ConfirmationEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Conversation (full form, returned by `GET /api/conversations/{id}`)

```json
{
  "conversationId": "conv-a4f…",
  "lineUserId": "Uabcdef0123456789abcdef0123456789",
  "originalText": "Email the project update to bob@example.com",
  "status": "AWAITING_CONFIRMATION",
  "plan": {
    "intent": "SEND_EMAIL",
    "reasoning": "User requested email to a named recipient.",
    "calendarParams": null,
    "gmailParams": {
      "recipientEmail": "bob@example.com",
      "subject": "Project update",
      "body": "Please find the project update as discussed.",
      "ccEmail": null
    },
    "clarificationText": null
  },
  "result": null,
  "blockerReason": null,
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": null
}
```

### ConversationRow (list form, returned by `GET /api/conversations`)

Same fields as `Conversation` minus `plan.gmailParams.body` (truncated to 120 chars) and `result.summary` (truncated to 120 chars). The UI fetches the full conversation by id on row expand via `GET /api/conversations/{id}`. Every nullable field declared `Optional<T>`.

### ConfirmationRequest (returned by `GET /api/confirmations/{id}`)

```json
{
  "conversationId": "conv-a4f…",
  "params": {
    "recipientEmail": "bob@example.com",
    "subject": "Project update",
    "body": "Please find the project update as discussed.",
    "ccEmail": null
  },
  "requestedAt": "2026-06-28T09:10:05Z",
  "respondedAt": null,
  "status": "PENDING"
}
```

### SSE event format

```
event: conversation-update
data: { "conversationId": "conv-a4f…", "status": "AWAITING_CONFIRMATION", ... }
```

One event per state transition. Clients reconcile by `conversationId`. The SSE channel also emits `event: confirmation-update` when a `ConfirmationEntity` changes status:

```
event: confirmation-update
data: { "conversationId": "conv-a4f…", "status": "APPROVED", "respondedAt": "2026-06-28T09:10:42Z" }
```
