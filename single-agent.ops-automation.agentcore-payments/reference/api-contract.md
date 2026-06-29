# API Contract: Agent-Initiated Payment Settlement

All endpoints are served from `http://localhost:9164`. No authentication is enforced in local development; deployers add auth at the gateway layer.

---

## POST /tasks

Submit a new payment task. The endpoint creates `PaymentTaskEntity` and starts `PaymentAgentSession`.

**Owning component:** `PaymentTaskEndpoint`

**Request body**

```json
{
  "taskId": "task-001",
  "description": "Fetch enriched firmographic data for accounts A, B, and C from the enrichment API",
  "budgetCents": 5000,
  "approvalThresholdCents": 1000
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `taskId` | string | Yes | Caller-assigned unique identifier |
| `description` | string | Yes | Natural-language task description for the agent |
| `budgetCents` | long | Yes | Total spend ceiling in cents |
| `approvalThresholdCents` | long | Yes | Per-payment approval threshold in cents |

**Response `201 Created`**

```json
{
  "taskId": "task-001",
  "status": "PENDING",
  "budgetCents": 5000,
  "approvalThresholdCents": 1000,
  "accumulatedSpentCents": 0,
  "createdAt": "2026-06-29T10:00:00Z"
}
```

**Response `409 Conflict`** — task with this `taskId` already exists.

---

## GET /tasks

List all tasks, newest first.

**Owning component:** `PaymentTaskEndpoint` (reads `PaymentLedgerView`)

**Response `200 OK`**

```json
[
  {
    "taskId": "task-001",
    "status": "ACTIVE",
    "budgetCents": 5000,
    "spentCents": 800,
    "remainingCents": 4200,
    "totalPayments": 1,
    "pendingApprovals": 0,
    "createdAt": "2026-06-29T10:00:00Z"
  }
]
```

---

## GET /tasks/{taskId}

Get a single task with full payment history.

**Owning component:** `PaymentTaskEndpoint` (reads `PaymentTaskEntity` directly)

**Response `200 OK`**

```json
{
  "taskId": "task-001",
  "description": "Fetch enriched firmographic data for accounts A, B, and C",
  "status": "AWAITING_APPROVAL",
  "budgetCents": 5000,
  "approvalThresholdCents": 1000,
  "accumulatedSpentCents": 800,
  "createdAt": "2026-06-29T10:00:00Z",
  "completedAt": null,
  "payments": [
    {
      "paymentId": "pay-001",
      "targetApi": "enrichment-api.example.com/firmographic",
      "amountCents": 800,
      "status": "SETTLED",
      "approvalRequired": false,
      "reviewedBy": null,
      "settledAt": "2026-06-29T10:00:05Z"
    },
    {
      "paymentId": "pay-002",
      "targetApi": "enrichment-api.example.com/deep-enrichment",
      "amountCents": 1500,
      "status": "PENDING_APPROVAL",
      "approvalRequired": true,
      "reviewedBy": null,
      "settledAt": null
    }
  ]
}
```

**Response `404 Not Found`** — task does not exist.

---

## GET /tasks/{taskId}/stream

Server-Sent Events stream for real-time task updates. Backed by `PaymentLedgerView`.

**Owning component:** `PaymentTaskEndpoint`

**SSE event format**

```
event: task-updated
data: {"taskId":"task-001","status":"AWAITING_APPROVAL","spentCents":800,"remainingCents":4200,"pendingApprovals":1}

event: payment-held
data: {"taskId":"task-001","paymentId":"pay-002","amountCents":1500,"targetApi":"enrichment-api.example.com/deep-enrichment"}

event: payment-approved
data: {"taskId":"task-001","paymentId":"pay-002","reviewedBy":"ops@example.com","decidedAt":"2026-06-29T10:05:00Z"}

event: budget-cap-reached
data: {"taskId":"task-001","spentCents":5000,"budgetCents":5000}

event: task-completed
data: {"taskId":"task-001","spentCents":4800,"totalPayments":5}
```

Heartbeat: `: keep-alive` comment line every 15 seconds.

---

## POST /tasks/{taskId}/approvals

Submit a human approval decision for a held payment.

**Owning component:** `PaymentTaskEndpoint` (routes to `PaymentApprovalWorkflow`)

**Request body**

```json
{
  "paymentId": "pay-002",
  "decision": "APPROVE",
  "reviewer": "ops@example.com",
  "note": "Approved — within Q3 enrichment budget."
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `paymentId` | string | Yes | Payment being decided |
| `decision` | string | Yes | `APPROVE` or `REJECT` |
| `reviewer` | string | Yes | Reviewer identity |
| `note` | string | No | Optional reviewer note |

**Response `200 OK`**

```json
{
  "paymentId": "pay-002",
  "decision": "APPROVE",
  "reviewer": "ops@example.com",
  "decidedAt": "2026-06-29T10:05:00Z"
}
```

**Response `404 Not Found`** — payment not found or not in `PENDING_APPROVAL` state.

**Response `409 Conflict`** — decision already recorded for this payment.

---

## GET /metadata/readme

Serves `README.md` as `text/markdown`.

**Owning component:** `PaymentTaskEndpoint`

---

## GET /metadata/risk-survey

Serves `risk-survey.yaml` as `text/yaml`.

**Owning component:** `PaymentTaskEndpoint`

---

## GET /metadata/eval-matrix

Serves `eval-matrix.yaml` as `text/yaml`.

**Owning component:** `PaymentTaskEndpoint`

---

## GET / and GET /app/*

Serves the static UI from classpath resources.

**Owning component:** `PaymentTaskEndpoint`
