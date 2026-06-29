# API Contract — Lead Score HITL

All endpoints are JSON unless noted. ACL is open to the internet (local-dev only). The `/api/*` surface is served by `LeadEndpoint`; `/` and `/app/*` by `AppEndpoint`.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/leads` | `{ "rawSource": "string", "company": "string", "contactName": "string" }` | `{ "leadId": "uuid" }` | LeadEndpoint → InboundLeadQueue |
| POST | `/api/leads/{leadId}/approve` | `{ "reviewedBy": "string", "comment": "string" }` | `200` / `404` | LeadEndpoint → LeadEntity |
| POST | `/api/leads/{leadId}/reject` | `{ "reviewedBy": "string", "reason": "string" }` | `200` / `404` | LeadEndpoint → LeadEntity |
| GET | `/api/leads?status=...` | — | `{ "leads": [Lead, ...] }` | LeadEndpoint → LeadsView (client-side filter) |
| GET | `/api/leads/{leadId}` | — | `Lead` / `404` | LeadEndpoint → LeadsView |
| GET | `/api/leads/sse` | — | SSE stream of `Lead` | LeadEndpoint → LeadsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | LeadEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | LeadEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | LeadEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

### Lead (response form)

Lifecycle fields are nullable; serialized as the raw value or `null` (Akka serializes `Optional<T>` transparently — Lesson 6).

```json
{
  "id": "uuid",
  "company": "string or null",
  "contactName": "string or null (sanitized)",
  "rawSource": "string or null",
  "status": "NEW | COLLECTED | ANALYZED | SCORED | SHORTLISTED | APPROVED | REJECTED | CONTACTED",
  "collectedAt": "ISO-8601 or null",
  "enrichedProfile": "string or null",
  "analyzedAt": "ISO-8601 or null",
  "analysisSummary": "string or null",
  "scoredAt": "ISO-8601 or null",
  "score": "0-100 or null",
  "scoreRationale": "string or null",
  "shortlisted": "boolean or null",
  "reviewedAt": "ISO-8601 or null",
  "reviewedBy": "string or null",
  "reviewDecision": "APPROVED | REJECTED | null",
  "reviewComment": "string or null",
  "outreachDraftedAt": "ISO-8601 or null",
  "outreachSubject": "string or null",
  "outreachBody": "string or null"
}
```

### Submit request

```json
{ "rawSource": "Acme Robotics — referral from trade show", "company": "Acme Robotics", "contactName": "Jordan Lee" }
```

`contactName` is sanitized by `PiiSanitizer` before storage; the raw value is never persisted or sent to a model.

## SSE event format

`GET /api/leads/sse` emits one event per lead change:

```
event: lead
data: { ...Lead JSON as above... }

```

The browser updates the matching row in place by `id`.
