# API contract — antom-payment

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/payments` | `SubmitPaymentRequest` | `201 { paymentId }` | `PaymentEndpoint` → `PaymentEntity`, `PaymentWorkflow` |
| `GET` | `/api/payments` | — | `200 [ Payment... ]` (newest-first) | `PaymentEndpoint` ← `PaymentView` |
| `GET` | `/api/payments/{id}` | — | `200 Payment` / `404` | `PaymentEndpoint` ← `PaymentView` |
| `POST` | `/api/payments/{id}/approve` | — | `200 { paymentId, status: "EXECUTING" }` / `409` | `PaymentEndpoint` → `PaymentWorkflow` |
| `POST` | `/api/payments/{id}/deny` | — | `200 { paymentId, status: "REJECTED" }` / `409` | `PaymentEndpoint` → `PaymentWorkflow` |
| `GET` | `/api/payments/sse` | — | `text/event-stream` | `PaymentEndpoint` ← `PaymentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPaymentRequest (request body)

```json
{
  "type": "INITIATE",
  "currency": "USD",
  "amountMinorUnits": 500000,
  "recipientId": "antom-merchant-0042",
  "method": "BANK_TRANSFER",
  "memo": "Q2 subscription renewal — invoice 2026-Q2-0187",
  "submittedBy": "operator-jane"
}
```

### Payment (response body — full state)

```json
{
  "paymentId": "pay-7d3a...",
  "instruction": {
    "paymentId": "pay-7d3a...",
    "type": "INITIATE",
    "currency": "USD",
    "amountMinorUnits": 500000,
    "recipientId": "antom-merchant-0042",
    "method": "BANK_TRANSFER",
    "memo": "Q2 subscription renewal — invoice 2026-Q2-0187",
    "submittedBy": "operator-jane",
    "submittedAt": "2026-06-28T14:05:00Z"
  },
  "result": {
    "outcome": "settled",
    "apiResponse": {
      "transactionId": "TX-9f3a2b...",
      "antomStatus": "SUCCESS",
      "settledAmountMinorUnits": 500000,
      "feeMinorUnits": 250,
      "currency": "USD",
      "settledAt": "2026-06-28T14:05:28Z"
    },
    "agentSummary": "Initiated a 5 000.00 USD bank transfer to antom-merchant-0042; Antom returned SUCCESS with transaction TX-9f3a2b.",
    "decidedAt": "2026-06-28T14:05:29Z"
  },
  "fraudSignal": null,
  "status": "SETTLED",
  "createdAt": "2026-06-28T14:05:00Z",
  "finishedAt": "2026-06-28T14:05:29Z"
}
```

### Payment (response body — awaiting approval)

```json
{
  "paymentId": "pay-8c1f...",
  "instruction": {
    "type": "INITIATE",
    "currency": "EUR",
    "amountMinorUnits": 1500000,
    "recipientId": "antom-merchant-0099",
    "method": "CARD",
    "memo": "Annual license fee",
    "submittedBy": "operator-alex",
    "submittedAt": "2026-06-28T15:00:00Z"
  },
  "result": null,
  "fraudSignal": null,
  "status": "AWAITING_APPROVAL",
  "createdAt": "2026-06-28T15:00:00Z",
  "finishedAt": null
}
```

### Payment (response body — halted)

```json
{
  "paymentId": "pay-3b9e...",
  "instruction": { "...": "..." },
  "result": null,
  "fraudSignal": {
    "signalType": "velocity-breach",
    "detail": "3 payments to same recipient within 60 seconds",
    "detectedAt": "2026-06-28T14:07:11Z"
  },
  "status": "HALTED",
  "createdAt": "2026-06-28T14:07:00Z",
  "finishedAt": "2026-06-28T14:07:11Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: payment-update
data: { "paymentId": "pay-7d3a...", "status": "SETTLED", "result": { ... }, ... }
```

One event per state transition (`REQUESTED`, `AUTHORIZED`, `REJECTED`, `AWAITING_APPROVAL`, `EXECUTING`, `SETTLED`, `QUERIED`, `HALTED`, `FAILED`). Clients reconcile by `paymentId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). The `submittedBy` field is taken from the request body in the blueprint; a deployer adding identity must extract it from an authenticated session token and remove it from the request body. The `POST /api/payments/{id}/approve` and `/deny` endpoints must additionally be restricted to operators with the payment-approval role — the blueprint leaves this wiring as `TO_BE_COMPLETED_BY_DEPLOYER` in `risk-survey.yaml`.
