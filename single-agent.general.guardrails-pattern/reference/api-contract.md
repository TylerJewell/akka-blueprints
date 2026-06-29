# API contract — guardrails-pattern

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/interactions` | `SubmitInteractionRequest` | `201 { interactionId }` | `InteractionEndpoint` → `InteractionEntity` + `InteractionWorkflow` |
| `GET` | `/api/interactions` | — | `200 [ Interaction... ]` (newest-first) | `InteractionEndpoint` ← `InteractionView` |
| `GET` | `/api/interactions/{id}` | — | `200 Interaction` / `404` | `InteractionEndpoint` ← `InteractionView` |
| `GET` | `/api/interactions/sse` | — | `text/event-stream` | `InteractionEndpoint` ← `InteractionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitInteractionRequest (request body)

```json
{
  "promptText": "What is the boiling point of water at 3000 m altitude?",
  "policyProfile": "GENERAL",
  "submittedBy": "user-42"
}
```

`policyProfile` must be one of `GENERAL`, `CONSERVATIVE`, `STRICT`. Omitting it defaults to `GENERAL`.

### Interaction (response body)

```json
{
  "interactionId": "i-7ac...",
  "request": {
    "interactionId": "i-7ac...",
    "promptText": "What is the boiling point of water at 3000 m altitude?",
    "policyProfile": "GENERAL",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T12:00:00Z"
  },
  "inputGuardResult": {
    "passed": true,
    "ruleId": null,
    "reason": null,
    "checkedAt": "2026-06-28T12:00:00.050Z"
  },
  "reply": {
    "category": "FACTUAL",
    "text": "At approximately 3000 m above sea level, water boils at around 90 °C ...",
    "confidence": 0.92,
    "citations": ["Engineering Toolbox: Boiling point of water at altitude"],
    "repliedAt": "2026-06-28T12:00:05Z"
  },
  "outputGuardResult": {
    "passed": true,
    "ruleId": null,
    "reason": null,
    "checkedAt": "2026-06-28T12:00:05.020Z"
  },
  "status": "REPLY_RECORDED",
  "createdAt": "2026-06-28T12:00:00Z",
  "finishedAt": "2026-06-28T12:00:05.025Z"
}
```

A blocked interaction (input guardrail fired):

```json
{
  "interactionId": "i-9ff...",
  "request": { "promptText": "...<blocked prompt>...", "policyProfile": "STRICT", "submittedBy": "user-5", "submittedAt": "2026-06-28T12:01:00Z" },
  "inputGuardResult": {
    "passed": false,
    "ruleId": "STRICT-jailbreak-marker-1",
    "reason": "Prompt matches jailbreak pattern: 'ignore previous instructions'",
    "checkedAt": "2026-06-28T12:01:00.030Z"
  },
  "reply": null,
  "outputGuardResult": null,
  "status": "BLOCKED",
  "createdAt": "2026-06-28T12:01:00Z",
  "finishedAt": "2026-06-28T12:01:00.035Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: interaction-update
data: { "interactionId": "i-7ac...", "status": "REPLY_RECORDED", "reply": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PROMPT_CHECKED`, `BLOCKED`, `REPLYING`, `REPLY_RECORDED`, `FAILED`). Clients reconcile by `interactionId`; each event carries the full row at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
