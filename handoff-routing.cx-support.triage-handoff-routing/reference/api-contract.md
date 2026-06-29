# API contract — triage-handoff-routing

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/turns` | — | `200 [ Conversation... ]` (newest-first); supports optional `?intent=…&status=…` filtered client-side | `HandoffEndpoint` ← `ConversationView` |
| `GET` | `/api/turns/{id}` | — | `200 Conversation` / `404` | `HandoffEndpoint` ← `ConversationView` |
| `POST` | `/api/turns` | `{ "sessionId": String, "channel": String, "rawText": String }` | `201 { "turnId": String }` | `HandoffEndpoint` → `ConversationQueue` |
| `POST` | `/api/turns/{id}/review` | `{ "reviewedBy": String, "note": String, "publish": boolean }` | `200 Conversation` (status now `RESOLVED` or `ESCALATED`) / `409` if not in a blocked state | `HandoffEndpoint` → `ConversationEntity` |
| `GET` | `/api/turns/sse` | — | `text/event-stream` | `HandoffEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundTurn (request body for `POST /api/turns`, minus `turnId` and `receivedAt`)

```json
{
  "sessionId": "sess-4a91f",
  "channel": "chat",
  "rawText": "I can't log in — my two-factor code isn't being accepted."
}
```

### Conversation

```json
{
  "turnId": "turn-7c4e2d1a",
  "inbound": {
    "turnId": "turn-7c4e2d1a",
    "sessionId": "sess-4a91f",
    "channel": "chat",
    "rawText": "I can't log in — my two-factor code isn't being accepted.",
    "receivedAt": "2026-06-28T09:12:00Z"
  },
  "normalised": {
    "normalisedText": "i can't log in my two-factor code isn't being accepted.",
    "languageCode": "en",
    "containsSensitiveData": false
  },
  "triage": {
    "intent": "ACCOUNT",
    "confidence": "high",
    "reason": "Authentication failure on account access."
  },
  "routing": {
    "allowed": true,
    "reason": "High-confidence ACCOUNT routing for an authentication issue.",
    "rubricVersion": "v1"
  },
  "reply": {
    "responseText": "To reset your two-factor authenticator, go to Security Settings...",
    "action": "ACCOUNT_UPDATED",
    "specialistTag": "account",
    "repliedAt": "2026-06-28T09:12:09Z"
  },
  "responseVerdict": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": "2026-06-28T09:12:10Z"
}
```

A routing-blocked turn has `routing.allowed = false`, `status = "ROUTING_BLOCKED"`, `reply = null`, `responseVerdict = null`, `finishedAt = null`. A response-blocked turn has `routing.allowed = true`, `responseVerdict.allowed = false`, `responseVerdict.violations` non-empty, `status = "RESPONSE_BLOCKED"`, `finishedAt = null`. An escalated turn has `triage.intent = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### TriageDecision (returned by `TriageAgent`, embedded in `Conversation.triage`)

```json
{ "intent": "ACCOUNT" | "PRODUCT" | "RETURNS" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### RoutingVerdict (returned by `RoutingGuardrail`, embedded in `Conversation.routing`)

```json
{ "allowed": true | false,
  "reason": "one short sentence",
  "rubricVersion": "v1" }
```

### Reply (returned by any specialist, embedded in `Conversation.reply`)

```json
{ "responseText": "2–4 short paragraphs",
  "action": "INFORMATION_PROVIDED" | "ACCOUNT_UPDATED" | "RETURN_INITIATED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "account" | "product" | "returns",
  "repliedAt": "ISO-8601" }
```

### ResponseVerdict (returned by `ResponseGuardrail`, embedded in `Conversation.responseVerdict`)

```json
{ "allowed": true | false,
  "violations": ["invented-policy-citation", "return-window-outside-policy", …],
  "rubricVersion": "v1" }
```

## SSE event format

```
event: turn-update
data: { "turnId": "turn-…", "status": "TRIAGED", … full Conversation JSON … }
```

One event per state transition on the `ConversationView`. Clients reconcile by `turnId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/turns`).
- `404` — unknown `turnId`.
- `409` — `review` requested on a turn not currently in `ROUTING_BLOCKED` or `RESPONSE_BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
