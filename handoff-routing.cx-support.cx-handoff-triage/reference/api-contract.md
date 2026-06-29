# API contract — cx-handoff-triage

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/conversations` | — | `200 [ Conversation... ]` (newest-first); optional `?category=SALES|ISSUES_REPAIRS|UNCLEAR&status=…` filtered client-side | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ChatEndpoint` ← `ConversationView` |
| `POST` | `/api/conversations` | `{ "channel": String, "openingMessage": String, "customerHandle": String }` | `201 { "conversationId": String }` | `ChatEndpoint` → `ConversationQueue` |
| `POST` | `/api/conversations/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Conversation` (status now `RESOLVED`) / `409` if not `BLOCKED` | `ChatEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundMessage (request body for `POST /api/conversations`, minus `conversationId` and `receivedAt`)

```json
{
  "channel": "chat",
  "openingMessage": "Hi, my screen cracked on its own after two days. I want a replacement. Order ORD-4821.",
  "customerHandle": "cust-6130"
}
```

### Conversation

```json
{
  "conversationId": "cv-3a9f12e0",
  "incoming": {
    "conversationId": "cv-3a9f12e0",
    "channel": "chat",
    "openingMessage": "Hi, my screen cracked on its own after two days. I want a replacement. Order ORD-4821.",
    "customerHandle": "cust-6130",
    "receivedAt": "2026-06-28T09:10:00Z"
  },
  "sanitized": {
    "redactedText": "Hi, my screen cracked on its own after two days. I want a replacement. Order [REDACTED-ORDER-ID].",
    "piiCategoriesFound": ["order-id"]
  },
  "triage": {
    "category": "ISSUES_REPAIRS",
    "confidence": "high",
    "reason": "Defect report with explicit replacement request."
  },
  "response": {
    "responseText": "Your screen defect is covered under the standard warranty...",
    "action": "REPLACEMENT_ARRANGED",
    "specialistTag": "issues-repairs",
    "resolvedAt": "2026-06-28T09:10:22Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:23Z"
}
```

A blocked conversation has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated conversation has `triage.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### TriageDecision (returned by `TriageAgent`, embedded in `Conversation.triage`)

```json
{ "category": "SALES" | "ISSUES_REPAIRS" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### SpecialistResponse (returned by either specialist, embedded in `Conversation.response`)

```json
{ "responseText": "3–5 short paragraphs",
  "action": "ORDER_PLACED" | "REFUND_INITIATED" | "REPLACEMENT_ARRANGED" | "INFO_PROVIDED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "sales" | "issues-repairs",
  "resolvedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ResponseGuardrail`, embedded in `Conversation.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-delivery-window", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

### ToolCallRequest (sent to `ToolCallGuardrail` before each tool execution)

```json
{
  "toolName": "processRefund",
  "conversationId": "cv-3a9f12e0",
  "parameters": { "orderId": "ORD-4821", "amount": 89.00, "reason": "defective-screen" }
}
```

### ToolCallVerdict (returned by `ToolCallGuardrail`)

```json
{ "allowed": true | false,
  "reason": "" | "refund-exceeds-ceiling" | "invalid-fulfillment-state" | "missing-required-parameter" | "unknown-tool" }
```

## SSE event format

```
event: conversation-update
data: { "conversationId": "cv-…", "status": "TRIAGED", … full Conversation JSON … }
```

One event per state transition on the `ConversationView`. Clients reconcile by `conversationId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/conversations`).
- `404` — unknown `conversationId`.
- `409` — `unblock` requested on a conversation not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
