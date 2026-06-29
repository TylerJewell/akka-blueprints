# API contract — global-kyc-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/cases` | `SubmitCaseRequest` | `201 { caseId }` | `KycEndpoint` → `KycCaseEntity` |
| `GET` | `/api/cases` | — | `200 [ KycCaseRecord... ]` (newest-first) | `KycEndpoint` ← `KycView` |
| `GET` | `/api/cases/{id}` | — | `200 KycCaseRecord` / `404` | `KycEndpoint` ← `KycView` |
| `POST` | `/api/cases/{id}/review` | `ReviewRequest` | `200 { caseId, status }` / `404` / `409` | `KycEndpoint` → `KycPipelineWorkflow` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `KycEndpoint` ← `KycView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `KycEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `KycEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `KycEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCaseRequest (request body)

```json
{
  "applicantId": "CORP-001",
  "jurisdiction": "GB",
  "documentTypes": ["PASSPORT"]
}
```

`documentTypes` is a list of `DocumentType` enum values (`PASSPORT`, `NATIONAL_ID`, `DRIVERS_LICENSE`, `RESIDENCE_PERMIT`). If omitted, `CollectTools.fetchDocument` attempts to fetch all available types for the applicant.

### ReviewRequest (request body — POST /api/cases/{id}/review)

```json
{
  "reviewerId": "officer@example.com",
  "resolution": "APPROVED_AS_SUBMITTED",
  "notes": "Document scans reviewed; identity confirmed."
}
```

`resolution` must be one of `APPROVED_AS_SUBMITTED` or `OVERRIDDEN_TO_PASS`. The endpoint returns `409 Conflict` if the case is not in `PENDING_REVIEW` status. The endpoint returns `404` if `caseId` does not exist.

### KycCaseRecord (response body)

```json
{
  "caseId": "c-9fa3b2...",
  "applicantId": "CORP-001",
  "documents": {
    "applicantId": "CORP-001",
    "documents": [
      {
        "documentId": "doc-001-passport",
        "applicantId": "CORP-001",
        "docType": "PASSPORT",
        "issuingJurisdiction": "GB",
        "fullName": "NAME-7f3a1b22",
        "dateOfBirth": "DOB-9c0e44d1",
        "documentNumber": "DOC-****-4567",
        "fetchedAt": "2026-06-28T09:00:00Z"
      }
    ],
    "applicableRules": [
      { "ruleId": "GB-AML-DOC-001", "jurisdiction": "GB", "description": "Primary photo ID required", "requiredDocumentType": "PASSPORT" },
      { "ruleId": "GB-AML-AGE-001", "jurisdiction": "GB", "description": "Applicant must be 18+", "requiredDocumentType": null }
    ],
    "collectedAt": "2026-06-28T09:00:00Z"
  },
  "verification": {
    "documentVerifications": [
      { "documentId": "doc-001-passport", "status": "VERIFIED", "statusReason": "Document format valid, not expired, hash consistent" }
    ],
    "ruleResults": [
      { "ruleId": "GB-AML-DOC-001", "passed": true, "failureReason": null },
      { "ruleId": "GB-AML-AGE-001", "passed": true, "failureReason": null }
    ],
    "verifiedAt": "2026-06-28T09:00:10Z"
  },
  "decision": {
    "outcome": "PASS",
    "jurisdiction": "GB",
    "citedRuleIds": ["GB-AML-DOC-001", "GB-AML-AGE-001"],
    "rationale": "Passport verified and all mandatory AML rules satisfied for jurisdiction GB.",
    "decidedAt": "2026-06-28T09:00:20Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Rule citation coverage, rule ID validity, document verification, and outcome validity all satisfied.",
    "evaluatedAt": "2026-06-28T09:00:21Z"
  },
  "review": null,
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:21Z"
}
```

All PII fields in `documents[].fullName`, `documents[].dateOfBirth`, and `documents[].documentNumber` are tokenised by `PiiSanitizer` before the row is written to `KycView`. The response body never contains raw PII values.

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `review` is `null` for PASS-outcome cases; it carries the full `ReviewResolution` record for cases that went through the HITL gate.

### DECLINE + review example

```json
{
  "caseId": "c-2b8e71...",
  "status": "EVALUATED",
  "decision": {
    "outcome": "DECLINE",
    "jurisdiction": "US-NY",
    "citedRuleIds": ["USNY-AML-SANCTION-001", "USNY-AML-DOC-002"],
    "rationale": "Rule USNY-AML-SANCTION-001 failed: applicant document could not be matched against sanctions list.",
    "decidedAt": "2026-06-28T09:05:15Z"
  },
  "review": {
    "reviewerId": "officer@example.com",
    "resolution": "APPROVED_AS_SUBMITTED",
    "notes": "Secondary review confirmed: sanctions match was a false positive.",
    "resolvedAt": "2026-06-28T09:12:44Z"
  },
  "eval": {
    "score": 4,
    "rationale": "All checks passed; rule citation coverage slightly thin but meets minimum.",
    "evaluatedAt": "2026-06-28T09:12:45Z"
  }
}
```

### SSE event format

```
event: case-update
data: { "caseId": "c-9fa3b2...", "status": "VERIFIED", "verification": { ... }, ... }
```

One event per state transition (`CREATED`, `COLLECTING`, `COLLECTED`, `VERIFYING`, `VERIFIED`, `DECIDING`, `DECIDED`, `PENDING_REVIEW`, `REVIEWED`, `EVALUATED`, `FAILED`) and one per `SanitizerApplied` audit event:

```
event: sanitizer-applied
data: { "caseId": "c-9fa3b2...", "fieldsRedacted": ["fullName", "dateOfBirth", "documentNumber"], "appliedAt": "..." }
```

Clients reconcile by `caseId`; an event always carries the full row at the moment of transition (with PII already tokenised), so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp `submitterId` and `reviewerId` from the authenticated principal — extend `SubmitCaseRequest` and `ReviewRequest` to carry them, and propagate to the matching entity events. The `POST /api/cases/{id}/review` endpoint must additionally verify that the `reviewerId` holds the `compliance-officer` role before resuming the workflow.
