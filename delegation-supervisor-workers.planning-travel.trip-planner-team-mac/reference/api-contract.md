# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/trips` | `{ "destination": "string", "startDate": "YYYY-MM-DD", "endDate": "YYYY-MM-DD", "travellerCount": number, "budgetUsd": number }` | `{ "tripId": "uuid" }` | `TripEndpoint` → `TripRequestQueue` |
| GET | `/api/trips` | — | `{ "trips": [TripRow, ...] }` | `TripEndpoint` → `TripView` |
| GET | `/api/trips?status=AWAITING_APPROVAL` | — | filtered list (client-side filter) | `TripEndpoint` |
| GET | `/api/trips/{id}` | — | `TripRow` or 404 | `TripEndpoint` |
| GET | `/api/trips/sse` | — | `text/event-stream` of `TripRow` | `TripEndpoint` → `TripView` |
| POST | `/api/trips/{id}/approve` | `{ "token": "string" }` | `{ "status": "CONFIRMED" }` | `TripEndpoint` → `ApprovalEntity` |
| POST | `/api/trips/{id}/reject` | `{ "token": "string", "reason": "string" }` | `{ "status": "REJECTED" }` | `TripEndpoint` → `ApprovalEntity` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TripEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TripEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TripEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/trips` request:

```json
{
  "destination": "Lisbon, Portugal",
  "startDate": "2025-04-10",
  "endDate": "2025-04-17",
  "travellerCount": 2,
  "budgetUsd": 3000
}
```

`TripRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "tripId": "uuid",
  "destination": "string",
  "startDate": "YYYY-MM-DD",
  "endDate": "YYYY-MM-DD",
  "travellerCount": 2,
  "budgetUsd": 3000.0,
  "requestedBy": "string",
  "status": "PLANNING | IN_PROGRESS | AWAITING_APPROVAL | CONFIRMED | REJECTED | DEGRADED | BLOCKED",
  "destinationNotes": {
    "climate": "...",
    "visaRequirements": "...",
    "highlights": ["..."],
    "localTips": ["..."],
    "researchedAt": "ISO-8601"
  },
  "itinerary": {
    "days": [
      { "day": 1, "title": "...", "description": "...", "activities": ["..."] }
    ],
    "overallTheme": "...",
    "draftedAt": "ISO-8601"
  },
  "bookingProposals": [
    {
      "type": "flight",
      "description": "...",
      "estimatedCostUsd": 450.0,
      "referenceCode": "AK-20250410-LHR-LIS-ECO",
      "preparedAt": "ISO-8601"
    }
  ],
  "plan": {
    "compilationNotes": "...",
    "compiledAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/trips/sse` emits one event per trip change:

```
event: trip
data: { ...TripRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `tripId`. No polling.

## Approval flow

After a trip reaches `AWAITING_APPROVAL`, the TripRow includes the approval `token` embedded in the plan object. The App UI passes this token back with the approve or reject call. Tokens are single-use and bound to the `tripId`; a mismatched or already-used token returns 409.
