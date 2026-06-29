# API contract — akka-handoff-routing

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/conversations` | — | `200 [ Conversation... ]` (newest-first); optional `?category=…&status=…` filtered client-side | `HandoffEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `HandoffEndpoint` ← `ConversationView` |
| `POST` | `/api/conversations` | `{ "customerId": String, "channel": String, "userMessage": String, "priorTurns": [String] }` | `201 { "conversationId": String }` | `HandoffEndpoint` → `ConversationQueue` |
| `POST` | `/api/conversations/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Conversation` (status now `RESOLVED`) / `409` if not `BLOCKED` | `HandoffEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `HandoffEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundTurn (request body for `POST /api/conversations`, minus `conversationId` and `receivedAt`)

```json
{
  "customerId": "cust-4421",
  "channel": "chat",
  "userMessage": "I was billed twice this month. My card ending in 4242 was charged on the 3rd and again on the 5th.",
  "priorTurns": []
}
```

### Conversation

```json
{
  "conversationId": "conv-7a3f19d0",
  "inbound": {
    "conversationId": "conv-7a3f19d0",
    "customerId": "cust-4421",
    "channel": "chat",
    "userMessage": "I was billed twice this month. My card ending in 4242 was charged on the 3rd and again on the 5th.",
    "priorTurns": [],
    "receivedAt": "2026-06-28T09:12:00Z"
  },
  "filtered": {
    "filteredMessage": "I was billed twice this month. My card ending in [REDACTED-CARD] was charged on the 3rd and again on the 5th.",
    "filteredPriorTurns": [],
    "piiCategoriesFound": ["card-number"]
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routing": {
    "category": "BILLING",
    "confidence": "high",
    "reason": "Explicit duplicate-charge complaint."
  },
  "reply": {
    "replyBody": "Your account shows two charges dated the 3rd and 5th of this month...",
    "action": "REFUND_INITIATED",
    "specialistTag": "billing",
    "repliedAt": "2026-06-28T09:12:22Z"
  },
  "handoffScore": {
    "score": 5,
    "rationale": "Category right; reason names the duplicate-charge signal directly.",
    "scoredAt": "2026-06-28T09:12:14Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": "2026-06-28T09:12:23Z"
}
```

A blocked conversation has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated conversation has `routing.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### FilteredContext (embedded in `Conversation.filtered`)

```json
{
  "filteredMessage": "I was billed twice this month. My card ending in [REDACTED-CARD] was charged...",
  "filteredPriorTurns": [],
  "piiCategoriesFound": ["card-number"]
}
```

### GuardrailVerdict (returned by `RoutingGuardrail`, embedded in `Conversation.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["prompt-injection-signal", "unredacted-email-in-context", …],
  "rubricVersion": "v1" }
```

### RoutingDecision (returned by `TriageAgent`, embedded in `Conversation.routing`)

```json
{ "category": "BILLING" | "TECHNICAL" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### Reply (returned by either specialist, embedded in `Conversation.reply`)

```json
{ "replyBody": "2–4 short paragraphs",
  "action": "REFUND_INITIATED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "INFO_PROVIDED" | "ESCALATED",
  "specialistTag": "billing" | "technical",
  "repliedAt": "ISO-8601" }
```

### HandoffScore (returned by `HandoffJudge`, embedded in `Conversation.handoffScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: conversation-update
data: { "conversationId": "conv-…", "status": "ROUTED_BILLING", … full Conversation JSON … }
```

One event per state transition on the `ConversationView`. Clients reconcile by `conversationId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/conversations`).
- `404` — unknown `conversationId`.
- `409` — `unblock` requested on a conversation not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
