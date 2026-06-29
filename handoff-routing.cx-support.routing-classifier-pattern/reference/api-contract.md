# API contract — routing-classifier-pattern

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/messages` | — | `200 [ Message... ]` (newest-first); supports optional `?route=…&status=…` filtered client-side | `RoutingEndpoint` ← `MessageView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `RoutingEndpoint` ← `MessageView` |
| `POST` | `/api/messages` | `{ "channel": String, "subject": String, "body": String }` | `201 { "messageId": String }` | `RoutingEndpoint` → `MessageQueue` |
| `POST` | `/api/messages/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Message` (status now `PUBLISHED`) / `409` if not `REPLY_BLOCKED` | `RoutingEndpoint` → `MessageEntity` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `RoutingEndpoint` ← `MessageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingMessage (request body for `POST /api/messages`, minus `messageId` and `receivedAt`)

```json
{
  "channel": "email",
  "subject": "Wrong charge on my account",
  "body": "You charged me twice for the same month. I need the duplicate charge reversed."
}
```

### Message

```json
{
  "messageId": "msg-4a9c12d7",
  "incoming": {
    "messageId": "msg-4a9c12d7",
    "channel": "email",
    "subject": "Wrong charge on my account",
    "body": "You charged me twice for the same month. I need the duplicate charge reversed.",
    "receivedAt": "2026-06-28T09:10:00Z"
  },
  "routeDecision": {
    "route": "REFUND",
    "confidence": "high",
    "reason": "Explicit duplicate-charge refund request."
  },
  "routeVerdict": {
    "approved": true,
    "rejectionReason": null
  },
  "reply": {
    "replySubject": "Re: Wrong charge on my account",
    "replyBody": "Thank you for reaching out. We can see the duplicate charge on your account and have initiated a refund. The credit will appear within the standard processing window...",
    "action": "REFUND_INITIATED",
    "specialistTag": "refund",
    "repliedAt": "2026-06-28T09:10:14Z"
  },
  "replyVerdict": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "abandonReason": null,
  "status": "PUBLISHED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:15Z"
}
```

A route-blocked message has `routeVerdict.approved = false`, `routeVerdict.rejectionReason` populated, `status = "ROUTE_BLOCKED"`, and no `reply` or `replyVerdict`. A reply-blocked message has `replyVerdict.allowed = false`, `replyVerdict.violations` non-empty, `status = "REPLY_BLOCKED"`, `finishedAt = null`. An abandoned message has `routeDecision.route = "UNROUTABLE"`, `abandonReason` populated, `status = "ABANDONED"`.

### RouteDecision (returned by `RoutingClassifier`, embedded in `Message.routeDecision`)

```json
{ "route": "GENERAL" | "REFUND" | "TECHNICAL" | "UNROUTABLE",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### RouteVerdict (returned by `RouteGuardrail`, embedded in `Message.routeVerdict`)

```json
{ "approved": true | false,
  "rejectionReason": "route-not-in-registry" | "route-confidence-below-threshold" | "route-is-unroutable" | null }
```

### Reply (returned by any specialist, embedded in `Message.reply`)

```json
{ "replySubject": "Re: …",
  "replyBody": "2–4 short paragraphs",
  "action": "INFO_PROVIDED" | "REFUND_INITIATED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "general" | "refund" | "technical",
  "repliedAt": "ISO-8601" }
```

### ReplyVerdict (returned by `ReplyGuardrail`, embedded in `Message.replyVerdict`)

```json
{ "allowed": true | false,
  "violations": ["invented-refund-date", "unverifiable-legal-claim", …],
  "rubricVersion": "v1" }
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
- `409` — `unblock` requested on a message not currently in `REPLY_BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
