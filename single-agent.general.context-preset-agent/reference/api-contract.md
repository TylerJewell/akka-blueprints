# API contract — context-preset-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/preset-requests` | `SubmitPresetRequest` | `201 { requestId }` | `PresetRequestEndpoint` → `PresetRequestEntity` |
| `GET` | `/api/preset-requests` | — | `200 [ PresetRequestRow... ]` (newest-first) | `PresetRequestEndpoint` ← `PresetRequestView` |
| `GET` | `/api/preset-requests/{id}` | — | `200 PresetRequestRow` / `404` | `PresetRequestEndpoint` ← `PresetRequestView` |
| `GET` | `/api/preset-requests/sse` | — | `text/event-stream` | `PresetRequestEndpoint` ← `PresetRequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPresetRequest (request body)

```json
{
  "environment": "prod",
  "role": "admin",
  "requestText": "Flush the CDN edge cache for region us-east-1",
  "submittedBy": "operator-007"
}
```

### PresetRequestRow (response body)

```json
{
  "requestId": "pr-4ac...",
  "request": {
    "requestId": "pr-4ac...",
    "environment": "prod",
    "role": "admin",
    "requestText": "Flush the CDN edge cache for region us-east-1",
    "submittedBy": "operator-007",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "resolvedPreset": {
    "presetId": "prod:admin",
    "environment": "prod",
    "role": "admin",
    "modelId": "claude-sonnet-4-6",
    "allowedTools": ["readStateTool", "adminActionTool"],
    "instructionAddendum": "You are operating in the production environment. Treat all state-modifying actions as irreversible. Confirm the target resource before calling adminActionTool."
  },
  "result": {
    "answerText": "Cache flush triggered successfully. The CDN edge cache for region us-east-1 has been invalidated.",
    "toolCallLog": [
      {
        "toolName": "adminActionTool",
        "status": "INVOKED",
        "inputSummary": "action=cache-flush, region=us-east-1",
        "outputSummary": "Flush completed; 3,241 entries invalidated.",
        "calledAt": "2026-06-28T14:00:05Z"
      }
    ],
    "completedAt": "2026-06-28T14:00:05Z"
  },
  "audit": {
    "requestId": "pr-4ac...",
    "presetId": "prod:admin",
    "toolCallCount": 1,
    "blockedToolCallCount": 0,
    "auditedAt": "2026-06-28T14:00:06Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:06Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: request-update
data: { "requestId": "pr-4ac...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PRESET_RESOLVED`, `EXECUTING`, `COMPLETED`, `FAILED`). Clients reconcile by `requestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `environment`, `role`, and `submittedBy` from the authenticated principal rather than the request body. The guardrail enforcement is server-side; it does not rely on the client supplying a trustworthy role.
