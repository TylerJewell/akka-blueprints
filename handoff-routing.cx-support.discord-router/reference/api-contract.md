# API contract — discord-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/messages` | — | `200 [ Message... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `DiscordEndpoint` ← `MessageView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `DiscordEndpoint` ← `MessageView` |
| `POST` | `/api/messages` | `{ "guildId": String, "channelName": String, "authorHandle": String, "content": String }` | `201 { "messageId": String }` | `DiscordEndpoint` → `MessageQueue` |
| `POST` | `/api/messages/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Message` (status now `PUBLISHED`) / `409` if not `BLOCKED` | `DiscordEndpoint` → `MessageEntity` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `DiscordEndpoint` ← `MessageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingMessage (request body for `POST /api/messages`, minus `messageId` and `receivedAt`)

```json
{
  "guildId": "guild-88192",
  "channelName": "tech-support",
  "authorHandle": "developer#0042",
  "content": "Getting a 401 on /v1/events since yesterday. Token is valid. Email: dev@example.com."
}
```

### Message

```json
{
  "messageId": "msg-4c7a29e1",
  "incoming": {
    "messageId": "msg-4c7a29e1",
    "guildId": "guild-88192",
    "channelName": "tech-support",
    "authorHandle": "developer#0042",
    "content": "Getting a 401 on /v1/events since yesterday. Token is valid. Email: dev@example.com.",
    "receivedAt": "2026-06-28T10:15:00Z"
  },
  "sanitized": {
    "redactedContent": "Getting a 401 on /v1/events since yesterday. Token is valid. Email: [REDACTED-EMAIL].",
    "channelName": "tech-support",
    "piiCategoriesFound": ["email"]
  },
  "routing": {
    "category": "TECHNICAL",
    "confidence": "high",
    "reason": "Production 401 error on a named endpoint."
  },
  "reply": {
    "replyBody": "Which environment are you hitting (staging or prod)? Also, are you sending the token as a Bearer header or as a query param?",
    "action": "FOLLOW_UP_REQUESTED",
    "specialistTag": "technical",
    "repliedAt": "2026-06-28T10:15:14Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Category right; reason names the specific technical signal.",
    "scoredAt": "2026-06-28T10:15:10Z"
  },
  "escalationReason": null,
  "status": "PUBLISHED",
  "createdAt": "2026-06-28T10:15:00Z",
  "finishedAt": "2026-06-28T10:15:15Z"
}
```

A blocked message has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated message has `routing.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### RoutingDecision (returned by `ClassifierAgent`, embedded in `Message.routing`)

```json
{ "category": "COMMUNITY" | "TECHNICAL" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### BotReply (returned by either specialist, embedded in `Message.reply`)

```json
{ "replyBody": "2–3 sentence reply",
  "action": "ANSWERED" | "DOCS_LINKED" | "FOLLOW_UP_REQUESTED" | "ACKNOWLEDGED" | "ESCALATED",
  "specialistTag": "community" | "technical",
  "repliedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ReplyGuardrail`, embedded in `Message.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["unapproved-external-link", "staff-impersonation", …],
  "rubricVersion": "v1" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Message.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: message-update
data: { "messageId": "msg-…", "status": "CLASSIFIED", … full Message JSON … }
```

One event per state transition on the `MessageView`. Clients reconcile by `messageId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/messages`).
- `404` — unknown `messageId`.
- `409` — `unblock` requested on a message not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
