# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/advisory` | `{ "companyName": "string", "sector": "string", "stage": "string", "problemStatement": "string" }` | `{ "sessionId": "uuid" }` | `AdvisoryEndpoint` → `ProfileQueue` |
| GET | `/api/advisory` | — | `{ "sessions": [AdvisorySessionRow, ...] }` | `AdvisoryEndpoint` → `AdvisoryView` |
| GET | `/api/advisory?status=SYNTHESISED` | — | filtered list (client-side filter) | `AdvisoryEndpoint` |
| GET | `/api/advisory/{id}` | — | `AdvisorySessionRow` or 404 | `AdvisoryEndpoint` |
| GET | `/api/advisory/sse` | — | `text/event-stream` of `AdvisorySessionRow` | `AdvisoryEndpoint` → `AdvisoryView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AdvisoryEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AdvisoryEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AdvisoryEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/advisory` request:

```json
{
  "companyName": "Pareto Labs",
  "sector": "B2B SaaS",
  "stage": "pre-seed",
  "problemStatement": "Engineering teams spend 40% of sprint time on unplanned incident triage with no visibility into root cause patterns."
}
```

`AdvisorySessionRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "sessionId": "uuid",
  "companyName": "string",
  "sector": "string",
  "status": "PLANNING | IN_PROGRESS | SYNTHESISED | DEGRADED | BLOCKED",
  "marketLandscape": {
    "competitors": [{ "name": "...", "positioning": "...", "differentiator": "..." }],
    "targetSegments": ["..."],
    "marketSizeSummary": "...",
    "researchedAt": "ISO-8601"
  },
  "gtmStrategy": {
    "channels": ["..."],
    "pricingSignal": "...",
    "launchSequence": ["..."],
    "strategyAt": "ISO-8601"
  },
  "contentPlan": {
    "pillars": [{ "theme": "...", "rationale": "..." }],
    "formats": ["..."],
    "cadenceRecommendation": "...",
    "plannedAt": "ISO-8601"
  },
  "roadmap": {
    "phases": [{ "name": "...", "goal": "...", "milestones": ["..."] }],
    "keyRisk": "...",
    "roadmappedAt": "ISO-8601"
  },
  "report": {
    "executiveSummary": "...",
    "guardrailVerdict": "ok",
    "synthesisedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/advisory/sse` emits one event per session change:

```
event: session
data: { ...AdvisorySessionRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `sessionId`. No polling.
