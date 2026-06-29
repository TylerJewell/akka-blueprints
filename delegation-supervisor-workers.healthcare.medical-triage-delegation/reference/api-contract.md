# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/triage` | `{ "question": "string" }` | `{ "caseId": "uuid" }` | `TriageEndpoint` → `CaseQueue` |
| GET | `/api/triage` | — | `{ "cases": [MedicalCaseRow, ...] }` | `TriageEndpoint` → `CaseView` |
| GET | `/api/triage?status=RESPONDED` | — | filtered list (client-side filter) | `TriageEndpoint` |
| GET | `/api/triage/{id}` | — | `MedicalCaseRow` or 404 | `TriageEndpoint` |
| GET | `/api/triage/sse` | — | `text/event-stream` of `MedicalCaseRow` | `TriageEndpoint` → `CaseView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TriageEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TriageEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TriageEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/triage` request:

```json
{ "question": "I have had a persistent dry cough and low-grade fever for three days." }
```

`MedicalCaseRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "caseId": "uuid",
  "question": "string",
  "status": "TRIAGING | IN_REVIEW | RESPONDED | DEGRADED | BLOCKED",
  "symptomsAssessment": {
    "identifiedSymptoms": ["persistent dry cough", "low-grade fever"],
    "differentialSummary": "string",
    "urgencyIndicator": "low | moderate | high | emergency",
    "assessedAt": "ISO-8601"
  },
  "medicationGuidance": {
    "relevantMedications": ["paracetamol for fever and pain relief"],
    "interactions": [],
    "contraindications": [],
    "guidanceAt": "ISO-8601"
  },
  "careRecommendation": {
    "recommendedPathway": "GP appointment",
    "rationale": "string",
    "nextSteps": ["string"],
    "recommendedAt": "ISO-8601"
  },
  "response": {
    "summary": "string",
    "guardrailVerdict": "ok",
    "synthesisedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "complianceScore": "1-5 or null",
  "complianceRationale": "string or null",
  "hotlPending": false,
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/triage/sse` emits one event per case change:

```
event: case
data: { ...MedicalCaseRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `caseId`. No polling.
