# API contract

Every endpoint the generated `NegotiationEndpoint` and `AppEndpoint` expose. All `/api/*` responses are JSON unless noted. ACL is open to the internet for local development only.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/negotiations` | `StartRequest` | `{ "negotiationId": "uuid" }` | `NegotiationEndpoint` → `InboundRequestQueue` |
| GET | `/api/negotiations` | — (optional `?status=`) | `{ "negotiations": [Negotiation, ...] }` | `NegotiationEndpoint` → `NegotiationsView` |
| GET | `/api/negotiations/{id}` | — | `Negotiation` or 404 | `NegotiationEndpoint` → `NegotiationsView` |
| GET | `/api/negotiations/sse` | — | `text/event-stream` of `Negotiation` | `NegotiationEndpoint` → `NegotiationsView` |
| POST | `/api/system/halt` | — | `{ "halted": true }` | `NegotiationEndpoint` → `SystemControl` |
| POST | `/api/system/resume` | — | `{ "halted": false }` | `NegotiationEndpoint` → `SystemControl` |
| GET | `/api/system/status` | — | `{ "halted": bool }` | `NegotiationEndpoint` → `SystemControl` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `NegotiationEndpoint` (classpath) |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `NegotiationEndpoint` (classpath) |
| GET | `/api/metadata/readme` | — | `text/markdown` | `NegotiationEndpoint` (classpath) |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` (static-resources) |

`GET /api/negotiations?status=...` filters client-side from `getAllNegotiations` — the view query carries no `WHERE status` clause because the runtime cannot auto-index the enum column (Lesson 2).

## Request payloads

```java
record StartRequest(
  String item,          // what is being negotiated
  double buyerBudget,   // buyer's hard ceiling
  double sellerFloor    // seller's hard floor
) {}
```

A well-formed `StartRequest` has `buyerBudget > 0` and `sellerFloor > 0`. When `buyerBudget < sellerFloor` the negotiation is valid but will conclude `NO_DEAL` — that is one of the acceptance journeys, not an error.

The system endpoints (`/api/system/halt`, `/api/system/resume`) take no body.

## Response payloads

### `Negotiation` (the view row, returned by list / single / SSE)

JSON form; `Optional<T>` fields serialize as the raw value or `null` (Akka's Jackson config unwraps `Optional`, Lesson 6):

```json
{
  "id": "uuid",
  "item": "string or null",
  "buyerBudget": 0.0,
  "sellerFloor": 0.0,
  "status": "CREATED | NEGOTIATING | CONCLUDED | ESCALATED",
  "currentRound": 0,
  "offers": [
    {
      "round": 1,
      "party": "BUYER | SELLER",
      "price": 0.0,
      "terms": "string",
      "rationale": "string",
      "accept": false,
      "at": "ISO-8601"
    }
  ],
  "latestBuyerPrice": 0.0,
  "latestSellerPrice": 0.0,
  "outcome": "CONVERGED | NO_DEAL or null",
  "finalPrice": 0.0,
  "finalTerms": "string or null",
  "startedAt": "ISO-8601 or null",
  "concludedAt": "ISO-8601 or null",
  "escalatedAt": "ISO-8601 or null",
  "outcomeScore": 0.0,
  "outcomeNotes": "string or null"
}
```

`buyerBudget`, `sellerFloor`, `latestBuyerPrice`, `latestSellerPrice`, `finalPrice`, and `outcomeScore` are `Optional<Double>` in Java and serialize as a number or `null`. `offers` is never null — an empty array before the first turn.

## SSE event format

`GET /api/negotiations/sse` streams the `NegotiationsView` through `componentClient.forView().method(NegotiationsView::streamAllNegotiations).source()` mapped to Server-Sent Events. Each event's `data` field is one `Negotiation` JSON object (the same form above), emitted whenever any negotiation's projected row changes:

```
event: negotiation
data: {"id":"...","status":"NEGOTIATING","currentRound":2,"offers":[...], ...}

event: negotiation
data: {"id":"...","status":"CONCLUDED","outcome":"CONVERGED","finalPrice":420.0, ...}
```

The browser keeps a map keyed by `id` and replaces a row on each event, so a negotiation visibly advances round by round and then shows its concluded outcome and score.

## Metadata endpoints

The three `/api/metadata/*` endpoints read the corresponding file from `src/main/resources/metadata/` and return it with the matching content type (`text/yaml` or `text/markdown`). They power the Risk Survey, Eval Matrix, and Overview tabs without the browser needing to reach outside the service origin.
