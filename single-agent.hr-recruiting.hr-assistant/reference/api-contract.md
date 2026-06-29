# API contract — hr-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ HrQuery... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 HrQuery` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "rawQueryText": "How many days of PTO do I have left this year, and do unused days roll over?",
  "employeeId": "EM-001"
}
```

`employeeId` is optional. When omitted, the agent operates on the query text alone without a profile context.

### HrQuery (response body)

```json
{
  "queryId": "q-7c3...",
  "request": {
    "queryId": "q-7c3...",
    "rawQueryText": "How many days of PTO do I have left this year, and do unused days roll over?",
    "employeeId": "EM-001",
    "submittedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedQueryText": "How many days of PTO do I have left this year, and do unused days roll over?",
    "piiCategoriesFound": [],
    "specialCategoriesFound": [],
    "employeeProfile": {
      "employeeId": "EM-001",
      "department": "Engineering",
      "jobTitle": "Senior Software Engineer",
      "hireDate": "2023-03-01T00:00:00Z",
      "workLocation": "London, UK"
    }
  },
  "answer": {
    "applicabilityVerdict": "APPLICABLE",
    "answerText": "Full-time employees accrue 15 days of PTO annually at 1.25 days per month. Unused PTO balances up to 10 days carry over at year end; amounts above 10 days are forfeited. Your current balance is available in the HR portal under My Time.",
    "citations": [
      {
        "policyId": "pto-policy",
        "sectionReference": "Section 2.1",
        "quotedPassage": "Full-time employees earn 15 days of paid time off per calendar year, accrued monthly at 1.25 days per month beginning on the first day of employment."
      },
      {
        "policyId": "pto-policy",
        "sectionReference": "Section 2.4",
        "quotedPassage": "Unused PTO balances above 10 days are not carried forward into the subsequent calendar year."
      }
    ],
    "answeredAt": "2026-06-28T09:15:22Z"
  },
  "status": "ANSWER_RECORDED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7c3...", "status": "ANSWER_RECORDED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `ANSWERING`, `ANSWER_RECORDED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `employeeId` from the authenticated principal's session rather than the request body.
