# API contract — per-tenant-agent-fleet

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/customers` | `{ "name": String, "email": String, "tier"?: String }` | `202 { "customerId": String }` | `FleetEndpoint` → `CustomerRegistry` |
| `GET` | `/api/customers` | — | `200 [ Customer... ]` | `FleetEndpoint` ← `FleetStatusView` |
| `GET` | `/api/customers?status=…` | — | `200 [ Customer... ]` (filtered client-side) | `FleetEndpoint` ← `FleetStatusView` |
| `GET` | `/api/customers/{id}` | — | `200 Customer` / `404` | `FleetEndpoint` ← `CustomerRegistry` |
| `GET` | `/api/customers/sse` | — | `text/event-stream` (one event per customer status change) | `FleetEndpoint` ← `FleetStatusView` |
| `POST` | `/api/customers/{id}/chat` | `{ "content": String }` | `200 { "reply": String, "messageId": String }` / `404` / `409` (customer not `READY`) | `FleetEndpoint` → `AgentFleetWorkflow` |
| `GET` | `/api/customers/{id}/chat` | — | `200 [ ChatTurn... ]` | `FleetEndpoint` ← `ChatHistoryView` |
| `GET` | `/api/customers/{id}/chat/sse` | — | `text/event-stream` (one event per chat turn) | `FleetEndpoint` ← `ChatHistoryView` |
| `GET` | `/api/fleet/health` | — | `200 [ FleetHealthSummary... ]` | `FleetEndpoint` ← `StaleAgentMonitor` (latest run) |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Customer (full)

```json
{
  "customerId": "c-42a",
  "name": "Aiko Tanaka",
  "email": "aiko@example.com",
  "tier": "PREMIUM",
  "status": "READY",
  "profile": {
    "customerId": "c-42a",
    "tier": "PREMIUM",
    "summary": "Aiko is a PREMIUM-tier customer whose account suggests active engagement.",
    "keyFacts": ["PREMIUM tier — priority support", "Account registered recently"],
    "interactionHints": ["use professional but warm tone", "lead with concrete next steps"]
  },
  "profiledAt": "2026-06-28T09:01:12Z",
  "readyAt": "2026-06-28T09:01:45Z",
  "registeredAt": "2026-06-28T09:00:00Z"
}
```

### CustomerRow (fleet board / SSE — condensed)

```json
{
  "customerId": "c-42a",
  "name": "Aiko Tanaka",
  "tier": "PREMIUM",
  "status": "READY",
  "profileSummary": "Aiko is a PREMIUM-tier customer whose account suggests active engagement.",
  "welcomeSent": true,
  "welcomeBlockReason": null,
  "chatTurnCount": 3,
  "lastInteractionAt": "2026-06-28T10:14:00Z",
  "latestEvalScore": 88,
  "registeredAt": "2026-06-28T09:00:00Z"
}
```

The `CustomerRow` in `FleetStatusView` drops the full `CustomerProfile.keyFacts` and `interactionHints` arrays — it keeps only the profile summary. Lifecycle fields (`profile`, `profiledAt`, `readyAt`, `welcomeBlockReason`, `lastInteractionAt`) are `Optional<T>` in Java and serialize as the raw value or `null` (Lesson 6).

### ChatTurn

```json
{
  "messageId": "msg-7f3",
  "customerContent": "Can you remind me what tier I'm on?",
  "agentReply": "You're on the PREMIUM tier, which includes priority support response times.",
  "turnAt": "2026-06-28T10:14:00Z"
}
```

### FleetHealthSummary

```json
{
  "customerId": "c-17b",
  "daysSinceLastInteraction": 9,
  "recommendation": "no contact in 9 days — consider outreach"
}
```

Returned by `GET /api/fleet/health` as a list. An empty list means no customers meet the staleness threshold at the last monitor run.

### SSE event format

```
event: customer-update
data: { "customerId": "c-42a", "status": "READY", "name": "Aiko Tanaka", ... }

event: chat-turn
data: { "customerId": "c-42a", "messageId": "msg-7f3", "agentReply": "...", "turnAt": "..." }
```

`GET /api/customers/sse` emits one `customer-update` per customer status transition. `GET /api/customers/{id}/chat/sse` emits one `chat-turn` per appended turn for that customer. Clients reconcile the fleet board by `customerId` and the chat panel by `messageId`.
