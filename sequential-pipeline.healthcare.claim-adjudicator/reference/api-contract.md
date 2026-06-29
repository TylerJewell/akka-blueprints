# API contract — claim-adjudication-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/claims` | `SubmitClaimRequest` | `201 { claimId }` | `ClaimEndpoint` → `ClaimEntity` |
| `GET` | `/api/claims` | — | `200 [ ClaimRecord... ]` (newest-first) | `ClaimEndpoint` ← `ClaimView` |
| `GET` | `/api/claims/{id}` | — | `200 ClaimRecord` / `404` | `ClaimEndpoint` ← `ClaimView` |
| `GET` | `/api/claims/sse` | — | `text/event-stream` | `ClaimEndpoint` ← `ClaimView` |
| `POST` | `/api/claims/{id}/review` | `ReviewActionRequest` | `204` / `404` / `409` | `ClaimEndpoint` → `ClaimEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitClaimRequest (request body)

```json
{
  "memberInfo": {
    "tokenisedMemberId": "mbr-a1b2c3d4",
    "planId": "PPO-100",
    "groupId": "GRP-5001"
  },
  "procedureCodes": [
    { "code": "80053", "system": "CPT", "description": "Comprehensive metabolic panel" }
  ],
  "serviceDate": "2026-06-15",
  "providerId": "npi-[REDACTED:npi:3f8a7c21]"
}
```

Note: `providerId` carries a pre-sanitised NPI token. The UI submits the raw NPI; `PhiSanitizerGuardrail` sanitises it before any tool body sees it. The API response and view records always carry the tokenised form.

### ReviewActionRequest (request body)

```json
{
  "action": "APPROVE_DENIAL",
  "reviewerId": "rev-jsmith",
  "notes": "Exclusion confirmed; pre-auth was not obtained."
}
```

`action` must be `"APPROVE_DENIAL"` or `"OVERRIDE_TO_APPROVE"`. Returns `409 Conflict` if the claim is not in `PENDING_REVIEW` status.

### ClaimRecord (response body)

```json
{
  "claimId": "c-7a3f9d...",
  "memberInfo": {
    "tokenisedMemberId": "mbr-a1b2c3d4",
    "planId": "PPO-100",
    "groupId": "GRP-5001"
  },
  "procedureCodes": [
    { "code": "80053", "system": "CPT", "description": "Comprehensive metabolic panel" }
  ],
  "serviceDate": "2026-06-15",
  "providerId": "npi-[REDACTED:npi:3f8a7c21]",
  "validation": {
    "eligibility": {
      "eligible": true,
      "planName": "PPO-100 Standard",
      "coverageType": "PPO",
      "priorAuthRequired": [],
      "checkedAt": "2026-06-28T10:00:05Z"
    },
    "codeVerifications": [
      { "code": "80053", "system": "CPT", "recognised": true, "procedureFamily": "lab" }
    ],
    "validationWarnings": [],
    "validatedAt": "2026-06-28T10:00:05Z"
  },
  "coverage": {
    "applicableRules": [
      { "ruleId": "LAB-R1", "description": "Comprehensive metabolic panel covered at 80%", "applicability": "lab", "exclusion": false },
      { "ruleId": "LAB-R2", "description": "In-network lab benefit applies", "applicability": "lab", "exclusion": false }
    ],
    "exclusionRules": [],
    "coverageAmount": { "currency": "USD", "coveredUnits": 1, "coveredPercent": 0.80, "memberResponsibility": 12.40 },
    "coverageSummary": "Lab panel covered at 80% under in-network benefit.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "decision": {
    "outcome": "APPROVED",
    "rationale": {
      "narrative": "CPT 80053 covered at 80% under LAB-R1 and LAB-R2. No exclusions apply.",
      "citedRuleIds": ["LAB-R1", "LAB-R2"]
    },
    "approvedAmount": { "currency": "USD", "coveredUnits": 1, "coveredPercent": 0.80, "memberResponsibility": 12.40 },
    "denialCode": null,
    "decidedAt": "2026-06-28T10:00:15Z"
  },
  "eval": {
    "score": 5,
    "rationale": "All checks passed: rule citation count ≥ 2, code provenance verified, outcome consistent, evidence completeness met.",
    "evaluatedAt": "2026-06-28T10:00:16Z"
  },
  "status": "ADJUDICATION_EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:16Z",
  "phiRedactions": [
    { "field": "memberId", "maskedValue": "[REDACTED:memberId:a1b2c3d4]", "redactedAt": "2026-06-28T10:00:01Z" },
    { "field": "npi", "maskedValue": "[REDACTED:npi:3f8a7c21]", "redactedAt": "2026-06-28T10:00:01Z" }
  ],
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `phiRedactions` and `guardrailRejections` are arrays — populated incrementally as events land.

### SSE event format

```
event: claim-update
data: { "claimId": "c-7a3f9d...", "status": "EVALUATED_COVERAGE", "coverage": { ... }, ... }
```

One event per status transition (`CREATED`, `VALIDATING`, `VALIDATED`, `EVALUATING`, `EVALUATED_COVERAGE`, `DECIDING`, `DECIDED`, `PENDING_REVIEW`, `APPROVED`, `DENIED`, `ADJUDICATION_EVALUATED`, `ESCALATED`, `FAILED`) and one per audit side-event:

```
event: claim-phi-redacted
data: { "claimId": "c-7a3f9d...", "field": "memberId", "maskedValue": "[REDACTED:memberId:a1b2c3d4]", "redactedAt": "..." }

event: claim-rejection
data: { "claimId": "c-7a3f9d...", "phase": "VALIDATE", "tool": "buildDecisionRationale", "reason": "phase-violation: ...", "rejectedAt": "..." }
```

Clients reconcile by `claimId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, stamp `submittedBy` from the authenticated principal on `SubmitClaimRequest`, and require an authenticated `reviewerId` on `ReviewActionRequest`. Extend `ClaimCreated` and `HumanReviewRequested` events to carry these fields.
