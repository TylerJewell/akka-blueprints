# API contract — persona-hot-reload

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/personas/push` | `PushPersonaRequest` | `201 { changeId }` | `PersonaEndpoint` → `PersonaEntity` |
| `GET` | `/api/personas` | — | `200 [ PersonaChange... ]` (newest-first) | `PersonaEndpoint` ← `PersonaView` |
| `GET` | `/api/personas/{id}` | — | `200 PersonaChange` / `404` | `PersonaEndpoint` ← `PersonaView` |
| `GET` | `/api/personas/current` | — | `200 PersonaChange` / `404` | `PersonaEndpoint` ← `PersonaView` |
| `GET` | `/api/personas/sse` | — | `text/event-stream` | `PersonaEndpoint` ← `PersonaView` |
| `POST` | `/api/personas/chat` | `ChatRequest` | `200 { responseText, changeId }` | `PersonaEndpoint` → `PersonaAgent` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### PushPersonaRequest (request body)

```json
{
  "agentRole": "Customer support assistant for SaaS billing inquiries",
  "agentGoal": "Resolve billing questions accurately and escalate account-level issues",
  "agentInstructions": "Answer only billing-related questions. Do not speculate on product roadmap. Escalate refund requests above $500 to a human agent. Keep responses under 200 words.",
  "agentModel": "claude-sonnet-4-6",
  "pushedBy": "operator-platform-12"
}
```

`agentModel` is optional. Omit it to keep the currently configured model.

### PersonaChange (response body)

```json
{
  "changeId": "c-8ae...",
  "snapshot": {
    "changeId": "c-8ae...",
    "payload": {
      "agentRole": "Customer support assistant for SaaS billing inquiries",
      "agentGoal": "Resolve billing questions accurately and escalate account-level issues",
      "agentInstructions": "Answer only billing-related questions...",
      "agentModel": "claude-sonnet-4-6"
    },
    "pushedBy": "operator-platform-12",
    "pushedAt": "2026-06-28T14:00:00Z",
    "rejectionReason": null
  },
  "revalidation": {
    "status": "PASS",
    "probeResults": [
      {
        "probeId": "assistant-p1",
        "question": "[PROBE] What is the refund policy for annual subscriptions?",
        "response": "Annual subscriptions are eligible for a prorated refund within 30 days...",
        "outcome": "PASS",
        "rationale": "Response stayed within billing scope and did not speculate."
      }
    ],
    "evaluatedAt": "2026-06-28T14:00:28Z"
  },
  "monitoring": {
    "forwardedCount": 3,
    "open": false,
    "openedAt": "2026-06-28T14:00:28Z",
    "closedAt": "2026-06-28T14:05:28Z"
  },
  "status": "MONITORED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:05:28Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### ChatRequest (request body)

```json
{
  "queryId": "q-1af...",
  "message": "I was charged twice for my subscription this month."
}
```

### ChatResponse (response body)

```json
{
  "responseText": "I can see that concern — let me look into the duplicate charge. Could you provide your account email so I can pull the billing records?",
  "changeId": "c-8ae..."
}
```

### SSE event format

```
event: persona-update
data: { "changeId": "c-8ae...", "status": "REVALIDATING", "snapshot": { ... }, ... }
```

One event per state transition (`VALIDATING`, `REJECTED`, `ACTIVATING`, `ACTIVE`, `REVALIDATING`, `MONITORED`, `FAILED`). Clients reconcile by `changeId`; an event always carries the full row at the moment of transition.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and restrict `POST /api/personas/push` to authorized operators. `pushedBy` should be set from the authenticated principal, not the request body.
