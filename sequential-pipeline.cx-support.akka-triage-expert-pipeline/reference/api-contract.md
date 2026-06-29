# API contract — triage-expert-multi-agent-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/cases` | `SubmitCaseRequest` | `201 { caseId }` | `SupportCaseEndpoint` → `SupportCaseEntity` |
| `GET` | `/api/cases` | — | `200 [ SupportCaseRecord... ]` (newest-first) | `SupportCaseEndpoint` ← `SupportCaseView` |
| `GET` | `/api/cases/{id}` | — | `200 SupportCaseRecord` / `404` | `SupportCaseEndpoint` ← `SupportCaseView` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `SupportCaseEndpoint` ← `SupportCaseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCaseRequest (request body)

```json
{
  "issueDescription": "I was charged twice for my subscription this month and cannot reach billing support."
}
```

### SupportCaseRecord (response body)

```json
{
  "caseId": "case-a3f7b2...",
  "issueDescription": "I was charged twice for my subscription this month and cannot reach billing support.",
  "customerInfo": {
    "customerId": "cust-001",
    "name": "Jordan Smith",
    "email": "jordan.smith@example.com",
    "accountId": "ACC-10042",
    "phone": "+1-555-0191",
    "issueDescription": "I was charged twice for my subscription this month and cannot reach billing support.",
    "category": "BILLING_DISPUTE",
    "urgency": "HIGH",
    "gatheredAt": "2026-06-28T14:00:05Z"
  },
  "issueSummary": {
    "caseId": "case-a3f7b2...",
    "category": "BILLING_DISPUTE",
    "urgency": "HIGH",
    "problemStatement": "Customer reports a duplicate charge for the current subscription month and has been unable to reach billing support through normal channels.",
    "keyFacts": [
      "Account type: Premium",
      "Two identical charges visible in account transaction history",
      "Issue duration: 2 days",
      "No response from billing support email"
    ],
    "summarizedAt": "2026-06-28T14:00:15Z"
  },
  "sanitizedSummary": {
    "caseId": "case-a3f7b2...",
    "category": "BILLING_DISPUTE",
    "urgency": "HIGH",
    "problemStatement": "[CUSTOMER_NAME] reports a duplicate charge for the current subscription month and has been unable to reach billing support through normal channels.",
    "keyFacts": [
      "Account type: Premium",
      "Two identical charges visible in account transaction history",
      "Issue duration: 2 days",
      "No response from billing support email"
    ],
    "redactedTokens": ["CUSTOMER_NAME"],
    "sanitizedAt": "2026-06-28T14:00:16Z"
  },
  "recommendation": {
    "caseId": "case-a3f7b2...",
    "category": "BILLING_DISPUTE",
    "guidanceText": "Verify the duplicate charge in the billing system and initiate a refund for the second charge. Ask the customer to allow 3–5 business days for the refund to appear. If the duplicate originated from an automatic renewal, disable the renewal temporarily while the account is audited. Escalate to the billing team lead if the charge cannot be traced in the transaction log.",
    "citedArticles": [
      { "articleId": "KB-0042", "title": "Handling duplicate subscription charges", "relevanceScore": 0.94 },
      { "articleId": "KB-0017", "title": "Refund policy and processing timelines", "relevanceScore": 0.81 }
    ],
    "requiresHumanEscalation": true,
    "composedAt": "2026-06-28T14:00:35Z"
  },
  "eval": {
    "score": 5,
    "rationale": "all checks passed",
    "evaluatedAt": "2026-06-28T14:00:35Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:35Z",
  "guardrailBlocks": [
    {
      "rule": "no-diagnostic-promise",
      "excerpt": "will fix the duplicate charge",
      "blockedAt": "2026-06-28T14:00:28Z"
    }
  ]
}
```

Any lifecycle field that has not yet been populated is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailBlocks` is an array — empty on the happy path, populated when the guardrail blocked a non-compliant recommendation.

Note: `customerInfo` contains raw PII; `sanitizedSummary` does not. A UI that renders both panels should clearly mark the customer info section as `[pre-sanitization]` and restrict access to authorized roles in a production deployment.

### SSE event format

```
event: case-update
data: { "caseId": "case-a3f7b2...", "status": "RECOMMENDED", "recommendation": { ... }, ... }
```

One event per state transition (`CREATED`, `GATHERING`, `GATHERED`, `SUMMARIZING`, `SUMMARIZED`, `SANITIZING`, `SANITIZED`, `RECOMMENDING`, `RECOMMENDED`, `EVALUATED`, `FAILED`) and one per `GuardrailBlocked` audit event:

```
event: case-guardrail-block
data: { "caseId": "case-a3f7b2...", "rule": "no-diagnostic-promise", "excerpt": "will fix the duplicate charge", "blockedAt": "2026-06-28T14:00:28Z" }
```

Clients reconcile by `caseId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `SupportCaseEndpoint` with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend `SubmitCaseRequest` and `CaseCreated` to carry it. The `customerInfo` section should be restricted to roles with PII-read authorization; `sanitizedSummary` is the safe view for general support tooling.
