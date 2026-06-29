# API contract — pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/briefings` | `SubmitBriefingRequest` | `201 { briefingId }` | `BriefingEndpoint` → `BriefingEntity` |
| `GET` | `/api/briefings` | — | `200 [ BriefingRecord... ]` (newest-first) | `BriefingEndpoint` ← `BriefingView` |
| `GET` | `/api/briefings/{id}` | — | `200 BriefingRecord` / `404` | `BriefingEndpoint` ← `BriefingView` |
| `GET` | `/api/briefings/sse` | — | `text/event-stream` | `BriefingEndpoint` ← `BriefingView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitBriefingRequest (request body)

```json
{
  "topic": "LLM token-cost trends"
}
```

### BriefingRecord (response body)

```json
{
  "briefingId": "b-5af9c2...",
  "topic": "LLM token-cost trends",
  "signals": {
    "signals": [
      {
        "source": "TokenWatch monthly index",
        "url": "https://example.org/tokenwatch/2026-05",
        "snippet": "Per-million-input-token prices on the four largest tiers fell 18% year-over-year through May 2026.",
        "capturedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "collectedAt": "2026-06-28T10:00:00Z"
  },
  "analysis": {
    "themes": [
      { "themeId": "input-prices-falling", "label": "Input-token prices falling", "claimIds": ["c-7f3a1b22"] }
    ],
    "claims": [
      { "claimId": "c-7f3a1b22", "text": "Input-token prices fell ~18% YoY through May 2026.", "supportingSource": "TokenWatch monthly index" }
    ],
    "analyzedAt": "2026-06-28T10:00:05Z"
  },
  "briefing": {
    "title": "LLM token-cost trends, mid-2026",
    "summary": "Input-token prices continue to fall ...",
    "sections": [
      {
        "themeId": "input-prices-falling",
        "heading": "Input-token prices falling",
        "body": "TokenWatch records ...",
        "sources": [{ "label": "TokenWatch monthly index", "url": "https://example.org/tokenwatch/2026-05" }]
      }
    ],
    "writtenAt": "2026-06-28T10:00:10Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Theme coverage, source attribution, source provenance, and section parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z",
  "guardrailRejections": [
    {
      "phase": "COLLECT",
      "tool": "formatSection",
      "reason": "phase-violation: formatSection requires status in {ANALYZED, REPORTING} with analysis present, saw COLLECTING",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

### SSE event format

```
event: briefing-update
data: { "briefingId": "b-5af9c2...", "status": "ANALYZED", "analysis": { ... }, ... }
```

One event per state transition (`CREATED`, `COLLECTING`, `COLLECTED`, `ANALYZING`, `ANALYZED`, `REPORTING`, `REPORTED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: briefing-rejection
data: { "briefingId": "b-5af9c2...", "phase": "COLLECT", "tool": "formatSection", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `briefingId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitBriefingRequest` record and the `BriefingCreated` event to carry it.
