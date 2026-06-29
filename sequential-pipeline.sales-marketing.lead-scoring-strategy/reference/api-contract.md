# API Contract — Lead Scoring Strategy

All endpoints are JSON unless noted. ACL is open to the internet (local-dev only). The `/api/*` surface is served by `LeadEndpoint`; `/` and `/app/*` by `AppEndpoint`.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/leads` | `{ "company": "string", "contactName": "string", "formResponses": "string" }` | `{ "leadId": "uuid" }` | LeadEndpoint → InboundLeadQueue |
| GET | `/api/leads?status=...` | — | `{ "leads": [Lead, ...] }` | LeadEndpoint → LeadsView (client-side filter) |
| GET | `/api/leads/{leadId}` | — | `Lead` / `404` | LeadEndpoint → LeadsView |
| GET | `/api/leads/sse` | — | SSE stream of `Lead` | LeadEndpoint → LeadsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | LeadEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | LeadEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | LeadEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

There is no approve/reject surface — the pipeline is linear with no human gate.

## Payload schemas

### Lead (response form)

Lifecycle fields are nullable; serialized as the raw value or `null` (Akka serializes `Optional<T>` transparently — Lesson 6).

```json
{
  "id": "uuid",
  "company": "string or null",
  "contactName": "string or null (sanitized)",
  "formResponses": "string or null (sanitized)",
  "status": "NEW | INTAKE | RESEARCHED | SCORED | COMPLETE",
  "intakeAt": "ISO-8601 or null",
  "intakeProfile": "string or null",
  "researchedAt": "ISO-8601 or null",
  "researchBrief": "string or null",
  "scoredAt": "ISO-8601 or null",
  "score": "0-100 or null",
  "scoreRationale": "string or null",
  "evalAt": "ISO-8601 or null",
  "evalScore": "0-100 or null",
  "evalFlags": "string or null (\"none\" or comma-joined)",
  "strategizedAt": "ISO-8601 or null",
  "engagementStrategy": "string or null"
}
```

### Submit request

```json
{ "company": "Acme Robotics", "contactName": "Jordan Lee", "formResponses": "Need warehouse automation, ~50 robots, budget Q3, email jordan@acme.example" }
```

`contactName` and `formResponses` are sanitized by `PiiSanitizer` before storage; the raw values are never persisted or sent to a model.

## SSE event format

`GET /api/leads/sse` emits one event per lead change:

```
event: lead
data: { ...Lead JSON as above... }

```

The browser updates the matching row in place by `id`. A scored lead may receive a later `ScoreEvaluated` update (eval fields populated) and then a `StrategyProduced` update (strategy populated, status `COMPLETE`).
