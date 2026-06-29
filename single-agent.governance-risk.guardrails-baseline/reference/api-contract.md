# API contract — guardrails-baseline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/moderations` | `SubmitModerationRequest` | `201 { moderationId }` | `ModerationEndpoint` → `ModerationEntity` |
| `GET` | `/api/moderations` | — | `200 [ Moderation... ]` (newest-first) | `ModerationEndpoint` ← `ModerationView` |
| `GET` | `/api/moderations/{id}` | — | `200 Moderation` / `404` | `ModerationEndpoint` ← `ModerationView` |
| `GET` | `/api/moderations/sse` | — | `text/event-stream` | `ModerationEndpoint` ← `ModerationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitModerationRequest (request body)

```json
{
  "messageTitle": "User message #4821 — finance channel",
  "rawMessage": "You should definitely buy XYZ stock right now, guaranteed 20% annual return...",
  "rules": [
    {
      "ruleId": "fin-investment-advice",
      "description": "Message must not contain unlicensed investment advice or specific security recommendations.",
      "category": "financial-advice"
    },
    {
      "ruleId": "fin-guaranteed-returns",
      "description": "Message must not assert guaranteed financial returns.",
      "category": "financial-advice"
    }
  ],
  "submittedBy": "trust-safety-operator-7"
}
```

### Moderation (response body)

```json
{
  "moderationId": "m-9f3...",
  "request": {
    "moderationId": "m-9f3...",
    "messageTitle": "User message #4821 — finance channel",
    "rawMessage": "(raw text preserved for audit)",
    "rules": [
      { "ruleId": "fin-investment-advice", "description": "...", "category": "financial-advice" }
    ],
    "submittedBy": "trust-safety-operator-7",
    "submittedAt": "2026-06-28T12:34:00Z"
  },
  "sanitized": {
    "redactedMessage": "You should definitely buy XYZ stock right now, guaranteed 20% annual return...",
    "piiCategoriesFound": []
  },
  "decision": {
    "verdict": "BLOCK",
    "rationale": "Message contains an explicit guaranteed-return claim and unlicensed investment advice.",
    "ruleResults": [
      {
        "ruleId": "fin-investment-advice",
        "action": "BLOCK",
        "confidence": 0.92,
        "explanation": "Explicit recommendation to 'buy XYZ stock' without licensed-adviser disclosure."
      },
      {
        "ruleId": "fin-guaranteed-returns",
        "action": "BLOCK",
        "confidence": 0.97,
        "explanation": "'guaranteed 20% annual return' is a prohibited guaranteed-return assertion."
      }
    ],
    "decidedAt": "2026-06-28T12:34:18Z"
  },
  "audit": {
    "score": 5,
    "note": "Every rule result has a non-empty explanation; verdict is consistent with rule actions; confidence values are in range.",
    "auditedAt": "2026-06-28T12:34:19Z"
  },
  "status": "AUDITED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: moderation-update
data: { "moderationId": "m-9f3...", "status": "DECISION_RECORDED", "decision": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `MODERATING`, `DECISION_RECORDED`, `AUDITED`, `FAILED`). Clients reconcile by `moderationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
