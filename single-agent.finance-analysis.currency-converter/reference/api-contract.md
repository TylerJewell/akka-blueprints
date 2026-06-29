# API contract — currency-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversions` | `SubmitConversionRequest` | `201 { conversionId }` | `ConversionEndpoint` → `ConversionEntity` |
| `GET` | `/api/conversions` | — | `200 [ Conversion... ]` (newest-first) | `ConversionEndpoint` ← `ConversionView` |
| `GET` | `/api/conversions/{id}` | — | `200 Conversion` / `404` | `ConversionEndpoint` ← `ConversionView` |
| `GET` | `/api/conversions/sse` | — | `text/event-stream` | `ConversionEndpoint` ← `ConversionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitConversionRequest (request body)

```json
{
  "fromCurrency": "USD",
  "toCurrency": "EUR",
  "amount": "1500.00",
  "snapshotLabel": "spot",
  "requestedBy": "treasury-desk-7"
}
```

`amount` is a decimal string to preserve precision across JSON serialisation. `snapshotLabel` is one of `spot`, `morning-fix`, `end-of-day`.

### Conversion (response body)

```json
{
  "conversionId": "c-9a3f...",
  "request": {
    "conversionId": "c-9a3f...",
    "fromCurrency": "USD",
    "toCurrency": "EUR",
    "amount": "1500.00",
    "snapshotLabel": "spot",
    "requestedBy": "treasury-desk-7",
    "requestedAt": "2026-06-28T12:00:00Z"
  },
  "rateSnapshot": {
    "label": "spot",
    "fromCurrency": "USD",
    "toCurrency": "EUR",
    "rate": "0.9231",
    "capturedAt": "2026-06-28T11:45:00Z",
    "source": "seed-file"
  },
  "result": {
    "convertedAmount": "1384.6500",
    "rateApplied": "0.9231",
    "fromCurrency": "USD",
    "toCurrency": "EUR",
    "confidenceNote": "Rate is from today's spot feed captured 15 minutes ago; confidence is high.",
    "marketContext": "USD/EUR is among the most liquid pairs globally. Current rate reflects recent monetary policy signals from the Federal Reserve and European Central Bank.",
    "decidedAt": "2026-06-28T12:00:22Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Rate snapshot is 15 minutes old; confidence note and market context are both populated.",
    "evaluatedAt": "2026-06-28T12:00:23Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T12:00:00Z",
  "finishedAt": "2026-06-28T12:00:23Z"
}
```

Any lifecycle field that has not yet occurred is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: conversion-update
data: { "conversionId": "c-9a3f...", "status": "RESULT_RECORDED", "result": { ... }, ... }
```

One event per state transition (`REQUESTED`, `RATE_ATTACHED`, `CONVERTING`, `RESULT_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `conversionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
