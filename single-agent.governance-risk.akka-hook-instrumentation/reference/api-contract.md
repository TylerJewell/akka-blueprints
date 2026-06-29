# API contract — hook-instrumentation

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/observations` | `SubmitObservationRequest` | `201 { observationId }` | `ObservationEndpoint` → `ObservationEntity` |
| `GET` | `/api/observations` | — | `200 [ Observation... ]` (newest-first) | `ObservationEndpoint` ← `ObservationView` |
| `GET` | `/api/observations/{id}` | — | `200 Observation` / `404` | `ObservationEndpoint` ← `ObservationView` |
| `GET` | `/api/observations/sse` | — | `text/event-stream` | `ObservationEndpoint` ← `ObservationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitObservationRequest (request body)

```json
{
  "taskDescription": "Summarize the Q2 performance report and highlight the top 3 risk indicators.",
  "personaId": "observer-standard",
  "availableTools": [
    {
      "toolName": "read-file",
      "description": "Read a named file from the in-process document store.",
      "allowedCallers": ["observer-standard"]
    },
    {
      "toolName": "extract-key-points",
      "description": "Extract key points from a text block using heuristic rules.",
      "allowedCallers": ["observer-standard"]
    },
    {
      "toolName": "format-output",
      "description": "Format structured data into a human-readable string.",
      "allowedCallers": ["observer-standard"]
    }
  ],
  "simulateBlockedTool": false,
  "submittedBy": "platform-operator-7"
}
```

### Observation (response body)

```json
{
  "observationId": "obs-3cf...",
  "request": {
    "observationId": "obs-3cf...",
    "taskDescription": "Summarize the Q2 performance report and highlight the top 3 risk indicators.",
    "personaId": "observer-standard",
    "availableTools": [
      { "toolName": "read-file", "description": "...", "allowedCallers": ["observer-standard"] }
    ],
    "simulateBlockedTool": false,
    "submittedBy": "platform-operator-7",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "hookConfig": {
    "blockedToolNames": ["raw-database-query"],
    "sensitivePatterns": ["(?i)password\\s*=\\s*\\S+", "Bearer\\s+[A-Za-z0-9._-]+"],
    "credentialPatterns": ["Bearer\\s+[A-Za-z0-9._-]+", "[A-Za-z0-9]{32,}"]
  },
  "outcome": {
    "outcome": "SUCCESS",
    "responseText": "The Q2 report shows stable revenue with three elevated risk indicators: supply-chain delay (HIGH), FX exposure (MEDIUM), and regulatory-filing lag (MEDIUM). Recommended actions are in the findings.",
    "hookLog": [
      {
        "entryId": "h1a2b3c4-0001",
        "hookPoint": "BEFORE_LLM_CALL",
        "toolNameOrPhase": "llm-invocation-1",
        "verdict": "PASS_THROUGH",
        "originalPayloadHash": "",
        "sanitizedPayload": "(prompt passed through without modification)",
        "rejectReason": "",
        "firedAt": "2026-06-28T14:00:02Z"
      },
      {
        "entryId": "h1a2b3c4-0002",
        "hookPoint": "BEFORE_TOOL_CALL",
        "toolNameOrPhase": "read-file",
        "verdict": "ALLOWED",
        "originalPayloadHash": "",
        "sanitizedPayload": "(input passed through)",
        "rejectReason": "",
        "firedAt": "2026-06-28T14:00:03Z"
      },
      {
        "entryId": "h1a2b3c4-0003",
        "hookPoint": "AFTER_TOOL_CALL",
        "toolNameOrPhase": "read-file",
        "verdict": "PASS_THROUGH",
        "originalPayloadHash": "",
        "sanitizedPayload": "(output passed through without redaction)",
        "rejectReason": "",
        "firedAt": "2026-06-28T14:00:04Z"
      }
    ],
    "completedAt": "2026-06-28T14:00:18Z"
  },
  "coverage": {
    "score": 5,
    "note": "All tool calls have matching before/after hook pairs; BEFORE_LLM_CALL fired for every LLM invocation.",
    "scoredAt": "2026-06-28T14:00:19Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: observation-update
data: { "observationId": "obs-3cf...", "status": "COMPLETED", "outcome": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `HOOK_CHAIN_READY`, `EXECUTING`, `COMPLETED`, `FAILED`). Clients reconcile by `observationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
