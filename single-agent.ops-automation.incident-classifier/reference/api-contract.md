# API contract — incident-classifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/incidents` | `SubmitIncidentRequest` | `201 { incidentId }` | `ClassificationEndpoint` → `IncidentEntity` |
| `GET` | `/api/incidents` | — | `200 [ Incident... ]` (newest-first) | `ClassificationEndpoint` ← `ClassificationView` |
| `GET` | `/api/incidents/{id}` | — | `200 Incident` / `404` | `ClassificationEndpoint` ← `ClassificationView` |
| `GET` | `/api/incidents/sse` | — | `text/event-stream` | `ClassificationEndpoint` ← `ClassificationView` |
| `GET` | `/api/incidents/accuracy` | — | `200 AccuracyWindow` | `ClassificationEndpoint` ← `ClassificationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitIncidentRequest (request body)

```json
{
  "shortDescription": "PROD-DB-01 connection timeouts from application tier",
  "longDescription": "Starting at approximately 14:00 UTC, the primary application server (APP-SRV-02) began reporting intermittent JDBC connection timeouts to PROD-DB-01. Connection pool exhaustion observed in application logs. Database host is reachable via ICMP. Database process is running. Suspect connection pool misconfiguration following last night's maintenance window.",
  "callerId": "ops-engineer-07",
  "priorityHint": "P2"
}
```

### Incident (response body)

```json
{
  "incidentId": "inc-7f3a...",
  "submission": {
    "incidentId": "inc-7f3a...",
    "shortDescription": "PROD-DB-01 connection timeouts from application tier",
    "longDescription": "Starting at approximately 14:00 UTC, the primary application server ...",
    "callerId": "ops-engineer-07",
    "priorityHint": "P2",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "scope": {
    "categoryCount": 6,
    "subcategoryCount": 28,
    "ciCount": 24,
    "validatedAt": "2026-06-28T14:00:01Z"
  },
  "classification": {
    "category": "Database",
    "subcategory": "Connectivity",
    "affectedCi": "PROD-DB-01",
    "rationale": "The description names PROD-DB-01 explicitly and reports JDBC connection timeouts from the app tier during peak load, matching Database / Connectivity.",
    "decidedAt": "2026-06-28T14:00:18Z"
  },
  "eval": {
    "categoryInTaxonomy": true,
    "subcategoryUnderCategory": true,
    "ciInRegistry": true,
    "score": 5,
    "rationale": "All three fields match the taxonomy exactly.",
    "evaluatedAt": "2026-06-28T14:00:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### AccuracyWindow (response body — `/api/incidents/accuracy`)

```json
{
  "windowDays": 7,
  "totalContributions": 42,
  "fullyCorrectCount": 38,
  "fullyCorrectRate": 0.905,
  "dailyCounts": [
    { "date": "2026-06-22", "total": 5, "fullyCorrect": 5 },
    { "date": "2026-06-23", "total": 7, "fullyCorrect": 6 },
    { "date": "2026-06-24", "total": 6, "fullyCorrect": 6 },
    { "date": "2026-06-25", "total": 8, "fullyCorrect": 7 },
    { "date": "2026-06-26", "total": 4, "fullyCorrect": 4 },
    { "date": "2026-06-27", "total": 6, "fullyCorrect": 5 },
    { "date": "2026-06-28", "total": 6, "fullyCorrect": 5 }
  ]
}
```

### SSE event format

```
event: incident-update
data: { "incidentId": "inc-7f3a...", "status": "CLASSIFICATION_RECORDED", "classification": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `TAXONOMY_VALIDATED`, `CLASSIFYING`, `CLASSIFICATION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `incidentId`; each event carries the full row at the moment of transition so a late-joining client does not need to replay history.

## Authorization

ACL: open to the local development runtime. A deployer adding identity must wrap the endpoint with their auth middleware and derive `callerId` from the authenticated principal rather than the request body.
