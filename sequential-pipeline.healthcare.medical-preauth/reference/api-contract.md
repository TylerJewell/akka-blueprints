# API contract — medical-preauth

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/auth-requests` | `SubmitAuthRequest` | `201 { requestId }` | `PreAuthEndpoint` → `PreAuthEntity` |
| `GET` | `/api/auth-requests` | — | `200 [ PreAuthRow... ]` (newest-first) | `PreAuthEndpoint` ← `PreAuthView` |
| `GET` | `/api/auth-requests/{id}` | — | `200 PreAuthRow` / `404` | `PreAuthEndpoint` ← `PreAuthView` |
| `GET` | `/api/auth-requests/sse` | — | `text/event-stream` | `PreAuthEndpoint` ← `PreAuthView` |
| `POST` | `/api/auth-requests/{id}/acknowledge` | `AcknowledgeRequest` | `200 { status }` | `PreAuthEndpoint` → `PreAuthEntity` → `PreAuthWorkflow` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAuthRequest (request body)

```json
{
  "memberId": "M-1234567",
  "procedureCode": "77080",
  "diagnosisCode": "M81.0",
  "treatingPhysicianNpi": "1234567890",
  "clinicalNotes": "Patient is a 67-year-old postmenopausal female with prior fragility fracture at L2."
}
```

`memberId` and `clinicalNotes` are PHI-bearing fields. `PhiSanitizer` replaces them with hashed tokens before the LLM sees them; the raw values are retained only in the service's own storage under HIPAA controls.

### AcknowledgeRequest (request body)

```json
{
  "reviewerNotes": "Rationale and cited articles reviewed. Denial is consistent with policy pol-ortho-003."
}
```

Only valid when the request is in `PENDING_HUMAN_REVIEW` status. Returns `409 Conflict` if the request is not in that status.

### PreAuthRow (response body)

```json
{
  "requestId": "req-8bc4a1...",
  "memberInfo": {
    "memberId": "M-1234567",
    "planId": "PLN-9900",
    "groupNumber": "GRP-0042",
    "activeEligibility": true
  },
  "procedureRequest": {
    "procedureCode": "77080",
    "diagnosisCode": "M81.0",
    "treatingPhysicianNpi": "1234567890",
    "requestingFacility": "Riverside Imaging Center",
    "clinicalNotes": "[PHI redacted for API response]"
  },
  "validationResult": {
    "eligibility": {
      "eligible": true,
      "planType": "PPO",
      "coverageTier": "tier-2",
      "verifiedAt": "2026-06-28T09:00:00Z"
    },
    "procedureCheck": {
      "codeRecognized": true,
      "codeDescription": "Dual-energy X-ray absorptiometry (DEXA), bone density study",
      "icd10Linked": true,
      "checkedAt": "2026-06-28T09:00:01Z"
    },
    "validationPassed": true,
    "validatedAt": "2026-06-28T09:00:01Z"
  },
  "policyReview": {
    "matchedArticles": [
      {
        "articleId": "pol-dexa-001",
        "title": "Coverage criteria for bone density testing",
        "criteriaText": "Member must be female aged 65+ or have documented risk factors for osteoporosis.",
        "policyVersion": "2026-01"
      }
    ],
    "criteriaEvaluation": {
      "criteria": [
        {
          "criterionId": "cr-age-risk",
          "description": "Member aged 65+ or documented risk factors",
          "met": true,
          "evidence": "Clinical notes reference postmenopausal status and prior fracture"
        }
      ],
      "metCount": 1,
      "totalCount": 1,
      "evaluatedAt": "2026-06-28T09:00:10Z"
    },
    "reviewedAt": "2026-06-28T09:00:10Z"
  },
  "determination": {
    "outcome": "APPROVED",
    "rationale": "All policy criteria under pol-dexa-001 are satisfied. The member's clinical notes document postmenopausal status and a prior fragility fracture, meeting the age-and-risk-factor criterion. DEXA bone density testing is covered under the member's PPO tier-2 plan.",
    "citedArticleIds": ["pol-dexa-001"],
    "coveredProcedureCodes": ["77080"],
    "determinedAt": "2026-06-28T09:00:15Z"
  },
  "eval": {
    "score": 5,
    "rationale": "All four clinical completeness checks passed.",
    "evaluatedAt": "2026-06-28T09:00:15Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:15Z",
  "sanitizationLog": [
    {
      "field": "memberId",
      "hashedToken": "a3f7b1c920d4",
      "tool": "checkMemberEligibility",
      "sanitizedAt": "2026-06-28T09:00:00Z"
    }
  ],
  "phaseRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `sanitizationLog` is always an array; it is empty on requests where no PHI fields appeared in tool call payloads. `phaseRejections` is always an array; it is empty on the happy path.

Note: `procedureRequest.clinicalNotes` is redacted in the API response. The raw text is retained only in the service's internal event log. Downstream consumers receive `[PHI redacted for API response]`.

### SSE event format

```
event: auth-request-update
data: { "requestId": "req-8bc4a1...", "status": "POLICY_REVIEWED", "policyReview": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `VALIDATING`, `VALIDATED`, `REVIEWING`, `POLICY_REVIEWED`, `DETERMINING`, `DETERMINED`, `PENDING_HUMAN_REVIEW`, `EVALUATED`, `ESCALATED`, `FAILED`) and one per audit side-event:

```
event: auth-request-sanitization
data: { "requestId": "req-8bc4a1...", "field": "memberId", "hashedToken": "a3f7b1c920d4", "tool": "checkMemberEligibility", "sanitizedAt": "..." }
```

```
event: auth-request-rejection
data: { "requestId": "req-8bc4a1...", "phase": "VALIDATE", "tool": "buildRationale", "reason": "phase-violation: buildRationale requires status in {POLICY_REVIEWED, DETERMINING}, saw VALIDATING", "rejectedAt": "..." }
```

Clients reconcile by `requestId`; every state-transition event carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, enforce role-based access (clinicians submit; reviewers acknowledge; auditors list), and stamp a `submittedBy` and `acknowledgedBy` field from the authenticated principal — extend `SubmitAuthRequest` / `AcknowledgeRequest` records and the corresponding events to carry them.
