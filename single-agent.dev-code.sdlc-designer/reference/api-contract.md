# API contract — sdlc-technical-designer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/designs` | `SubmitDesignRequest` | `201 { designId }` | `DesignEndpoint` → `DesignRequestEntity` |
| `GET` | `/api/designs` | — | `200 [ DesignRow... ]` (newest-first) | `DesignEndpoint` ← `DesignView` |
| `GET` | `/api/designs/{id}` | — | `200 DesignRequestState` / `404` | `DesignEndpoint` ← `DesignView` |
| `GET` | `/api/designs/sse` | — | `text/event-stream` | `DesignEndpoint` ← `DesignView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitDesignRequest (request body)

```json
{
  "featureTitle": "User notification service",
  "featureDescription": "Build a notification service that sends email and Slack messages triggered by domain events...",
  "projectContextKey": "event-driven",
  "constraints": [
    {
      "constraintId": "lang-java",
      "description": "Implementation must be in Java 21.",
      "category": "language"
    },
    {
      "constraintId": "no-external-queues",
      "description": "No external message brokers; use Akka's built-in event sourcing.",
      "category": "infra"
    }
  ],
  "requestedBy": "engineer-42"
}
```

### DesignRequestState (response body)

```json
{
  "designId": "d-7a3...",
  "request": {
    "designId": "d-7a3...",
    "featureTitle": "User notification service",
    "featureDescription": "Build a notification service...",
    "projectContextKey": "event-driven",
    "constraints": [
      { "constraintId": "lang-java", "description": "...", "category": "language" }
    ],
    "requestedBy": "engineer-42",
    "submittedAt": "2026-06-28T12:34:00Z"
  },
  "context": {
    "contextKey": "event-driven",
    "architecturePattern": "event-driven",
    "existingComponents": ["AuthService", "UserRegistry"],
    "preferredPatterns": ["cqrs", "event-sourcing"],
    "targetLanguage": "Java"
  },
  "proposal": {
    "components": [
      {
        "componentId": "notificationEntity",
        "componentKind": "EventSourcedEntity",
        "rationale": "Delivery state must survive restarts and support replay for auditing.",
        "dependsOn": []
      },
      {
        "componentId": "channelRouter",
        "componentKind": "Consumer",
        "rationale": "Incoming domain events arrive on a topic; a Consumer routes to channel handlers.",
        "dependsOn": ["notificationEntity"]
      }
    ],
    "dataModel": [
      {
        "entityName": "NotificationRecord",
        "fields": [
          { "fieldName": "notificationId", "fieldType": "String", "required": true, "purpose": "Stable identifier for idempotent delivery tracking." },
          { "fieldName": "recipientId", "fieldType": "String", "required": true, "purpose": "Target user or system." },
          { "fieldName": "sentAt", "fieldType": "Optional<Instant>", "required": false, "purpose": "Set when the channel confirms delivery." }
        ],
        "eventTypes": ["NotificationQueued", "NotificationSent", "NotificationFailed"]
      }
    ],
    "apiSurface": [
      {
        "method": "POST",
        "path": "/api/notifications",
        "requestSummary": "{ recipientId, channel, payload }",
        "responseSummary": "201 { notificationId }",
        "owningComponent": "notificationEndpoint"
      }
    ],
    "decisionLog": [
      {
        "decisionId": "esed-for-delivery-state",
        "componentId": "notificationEntity",
        "decision": "Use EventSourcedEntity to track per-notification delivery state.",
        "rationale": "Delivery state has natural lifecycle transitions and must be auditable. An EventSourcedEntity captures every transition without schema migrations.",
        "alternativesConsidered": ["KeyValueEntity (no event history)", "in-memory map (no persistence)"]
      }
    ],
    "executiveSummary": "The notification service reacts to domain events and delivers messages over multiple channels. An EventSourcedEntity tracks delivery state per recipient; a Consumer fan-out routes incoming events to the appropriate channel handlers. The design fits the event-driven context profile and respects the no-external-queues constraint.",
    "decidedAt": "2026-06-28T12:34:20Z"
  },
  "eval": {
    "score": 4,
    "rationale": "All rationale and purpose fields are non-empty; one decision-log entry lists only one alternative — stronger coverage would improve the score.",
    "evaluatedAt": "2026-06-28T12:34:21Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:21Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: design-update
data: { "designId": "d-7a3...", "status": "PROPOSAL_RECORDED", "proposal": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `CONTEXT_LOADED`, `DESIGNING`, `PROPOSAL_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `designId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
