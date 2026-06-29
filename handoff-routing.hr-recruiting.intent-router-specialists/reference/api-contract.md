# API contract — core-semantic-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first); supports optional `?domain=HR|FINANCE|AMBIGUOUS&status=…` filtered client-side | `RouterEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `RouterEndpoint` ← `QueryView` |
| `POST` | `/api/queries` | `{ "requesterId": String, "channel": String, "subject": String, "body": String }` | `201 { "queryId": String }` | `RouterEndpoint` → `QueryQueue` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `RouterEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingQuery (request body for `POST /api/queries`, minus `queryId` and `receivedAt`)

```json
{
  "requesterId": "emp-4471",
  "channel": "portal",
  "subject": "Vacation balance for Q3",
  "body": "Hi, I want to check how many vacation days I have left before booking a trip."
}
```

### Query

```json
{
  "queryId": "qr-7c3af12e",
  "incoming": {
    "queryId": "qr-7c3af12e",
    "requesterId": "emp-4471",
    "channel": "portal",
    "subject": "Vacation balance for Q3",
    "body": "Hi, I want to check how many vacation days I have left before booking a trip.",
    "receivedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedSubject": "Vacation balance for Q3",
    "redactedBody": "Hi, I want to check how many vacation days I have left before booking a trip.",
    "piiCategoriesFound": []
  },
  "routing": {
    "domain": "HR",
    "confidence": "high",
    "reason": "Leave-balance inquiry is an HR entitlement question."
  },
  "guardrail": {
    "authorized": true,
    "rejections": [],
    "rubricVersion": "v1"
  },
  "answer": {
    "answerBody": "Your remaining vacation entitlement is tracked in the HR portal under Time & Absence. Per the leave policy (HR Policy §4.2), unused days carry over up to the annual cap on January 1. To view your current balance, navigate to My Time → Leave Balances.\n\n— HR Support",
    "action": "POLICY_CITED",
    "specialistTag": "hr",
    "answeredAt": "2026-06-28T09:15:14Z"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Domain right; reason names the leave-balance signal directly.",
    "scoredAt": "2026-06-28T09:15:10Z"
  },
  "escalationReason": null,
  "status": "ANSWERED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:15Z"
}
```

A blocked query has `guardrail.authorized = false`, `guardrail.rejections` non-empty, `status = "BLOCKED"`, `finishedAt` set to the block time. An escalated query has `routing.domain = "AMBIGUOUS"`, `escalationReason` populated, `status = "ESCALATED"`.

### RoutingDecision (returned by `IntentRouterAgent`, embedded in `Query.routing`)

```json
{ "domain": "HR" | "FINANCE" | "AMBIGUOUS",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### GuardrailVerdict (returned by `RoutingGuardrail`, embedded in `Query.guardrail`)

```json
{ "authorized": true | false,
  "rejections": ["ambiguous-domain", "low-confidence", "unauthorized-channel", "unknown-domain"],
  "rubricVersion": "v1" }
```

### QueryAnswer (returned by either specialist, embedded in `Query.answer`)

```json
{ "answerBody": "2–4 short paragraphs in policy terms",
  "action": "POLICY_CITED" | "PROCESS_EXPLAINED" | "REFERRED" | "ESCALATED",
  "specialistTag": "hr" | "finance",
  "answeredAt": "ISO-8601" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Query.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: query-update
data: { "queryId": "qr-…", "status": "CLASSIFIED", … full Query JSON … }
```

One event per state transition on the `QueryView`. Clients reconcile by `queryId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/queries`).
- `404` — unknown `queryId`.
- `409` — operation attempted on a query in an incompatible state.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
