# API contract — ad-spend-payment-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/campaigns` | `CampaignBrief` (see below) | `202 { "campaignId": String }` | `CampaignEndpoint` → `CampaignQueue` |
| `GET` | `/api/campaigns` | — | `200 [ CampaignRow... ]` | `CampaignEndpoint` ← `CampaignView` |
| `GET` | `/api/campaigns/{id}` | — | `200 Campaign` / `404` | `CampaignEndpoint` ← `CampaignEntity` |
| `GET` | `/api/campaigns/sse` | — | `text/event-stream` (one event per campaign change) | `CampaignEndpoint` ← `CampaignView` |
| `GET` | `/api/campaigns/{id}/approvals` | — | `200 [ ApprovalRequest... ]` | `CampaignEndpoint` ← `ApprovalEntity` |
| `POST` | `/api/campaigns/{id}/approvals/{placementId}/approve` | `{ "approvedBy": String }` | `200 { "placementId": String, "status": "APPROVED" }` | `CampaignEndpoint` → `ApprovalEntity` |
| `POST` | `/api/campaigns/{id}/approvals/{placementId}/reject` | `{ "rejectedBy": String, "reason": String }` | `200 { "placementId": String, "status": "REJECTED" }` | `CampaignEndpoint` → `ApprovalEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `CampaignEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `CampaignEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `CampaignEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### CampaignBrief (request body for `POST /api/campaigns`)

```json
{
  "title": "AkkaStore Developer Banner Ad",
  "brandVoice": "direct, technical, no hype",
  "targetAudience": "JVM and distributed-systems developers",
  "adFormat": "banner-728x90",
  "budgetWei": 50000000000000000,
  "requestedBy": "marketing@example.com"
}
```

### Campaign (full form, returned by `GET /api/campaigns/{id}`)

```json
{
  "campaignId": "c-3a1…",
  "title": "AkkaStore Developer Banner Ad",
  "requestedBy": "marketing@example.com",
  "budgetWei": 50000000000000000,
  "status": "AWAITING_APPROVAL",
  "creativeLedger": {
    "facts": ["title: AkkaStore Developer Banner Ad", "target: developers", "format: banner-728x90"],
    "missing": [],
    "plan": [
      "Copywriter: draft headline and body for developer banner",
      "Copywriter: refine to match brand voice",
      "Payment: buy developer banner slot for 0.02 ETH"
    ],
    "currentDispatch": {
      "executor": "PAYMENT",
      "task": "buy developer banner slot via allow-listed network wallet",
      "rationale": "Copy is approved; proceed to purchase the slot.",
      "proposedSpendWei": 20000000000000000
    }
  },
  "paymentLedger": {
    "entries": [
      {
        "placementId": "pl-001",
        "task": "buy developer banner slot",
        "approvalStatus": "PENDING",
        "approvedBy": null,
        "approvedAt": null,
        "paymentResult": null,
        "recordedAt": "2026-06-28T10:15:22Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:14:55Z",
  "finishedAt": null
}
```

### Campaign (list form, returned by `GET /api/campaigns`)

The list form is `CampaignRow` — same fields as `Campaign`, but `creativeLedger.plan` is omitted, `paymentLedger.entries` is truncated to the last 3 entries plus `truncatedFromTotal: int`, and each `sanitizedContent` is capped at 240 characters. The UI fetches the full campaign by id on row expand.

### ApprovalRequest (returned in `GET /api/campaigns/{id}/approvals`)

```json
{
  "campaignId": "c-3a1…",
  "placementId": "pl-001",
  "task": "buy developer banner slot",
  "amountWei": 20000000000000000,
  "destinationWallet": "0xABCDEF1234567890ABCDEF1234567890ABCDEF12",
  "token": "ETH",
  "status": "PENDING",
  "approvedBy": null,
  "decidedAt": null
}
```

### SSE event formats

```
event: campaign-update
data: { "campaignId": "c-3a1…", "status": "AWAITING_APPROVAL", ... }
```

One event per state transition. Clients reconcile by `campaignId`. The SSE channel also emits:

```
event: approval-update
data: { "campaignId": "c-3a1…", "placementId": "pl-001", "status": "APPROVED", "approvedBy": "alice@example.com", "decidedAt": "2026-06-28T10:16:05Z" }
```

```
event: control-update
data: { "halted": true, "reason": "investigating on-chain anomaly", "haltedAt": "2026-06-28T10:18:31Z" }
```
