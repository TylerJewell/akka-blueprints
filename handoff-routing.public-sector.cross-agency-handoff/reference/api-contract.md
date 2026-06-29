# API contract — cross-agency-case-handoff-mesh

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/cases` | — | `200 [ Case... ]` (newest-first); optional `?segment=INTAKE|BENEFITS&status=…` filtered client-side | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/cases/{id}` | — | `200 Case` / `404` | `CaseEndpoint` ← `CaseView` |
| `POST` | `/api/cases` | `{ "constituentId": String, "programType": String, "geographicRegion": String, "summary": String, "fullPayload": String }` | `201 { "caseId": String }` | `CaseEndpoint` → `CaseInbox` |
| `POST` | `/api/cases/{id}/approve-handoff` | `{ "approvedBy": String, "note": String }` | `200 Case` (status now `HANDOFF_EMITTED` or `CLOSED`) / `409` if not `HANDOFF_PENDING` | `CaseEndpoint` → `CaseEntity` |
| `POST` | `/api/cases/{id}/reject-handoff` | `{ "rejectedBy": String, "reason": String }` | `200 Case` (status now `HANDOFF_REJECTED`) / `409` if not `HANDOFF_PENDING` | `CaseEndpoint` → `CaseEntity` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundCase (request body for `POST /api/cases`, minus `caseId` and `submittedAt`)

```json
{
  "constituentId": "CONST-4412",
  "programType": "food-assistance",
  "geographicRegion": "US-CA",
  "summary": "New applicant requesting SNAP benefits for household of three adults.",
  "fullPayload": "Name: [redacted by submitter]. DOB: 1985-03-17. Annual income: $28,400."
}
```

### Case

```json
{
  "caseId": "cs-7a3f91d2",
  "inbound": {
    "caseId": "cs-7a3f91d2",
    "constituentId": "CONST-4412",
    "programType": "food-assistance",
    "geographicRegion": "US-CA",
    "summary": "New applicant requesting SNAP benefits for household of three adults.",
    "fullPayload": "...raw with PII...",
    "submittedAt": "2026-06-29T09:00:00Z"
  },
  "scoped": {
    "caseId": "cs-7a3f91d2",
    "programType": "food-assistance",
    "geographicRegion": "US-CA",
    "redactedSummary": "New applicant requesting SNAP benefits for household of three adults.",
    "redactedPayload": "Name: [REDACTED-NAME]. DOB: [REDACTED-DOB]. Annual income: [REDACTED-INCOME].",
    "droppedFieldCategories": ["ssn", "date-of-birth", "income-figure", "address"]
  },
  "routing": {
    "segment": "INTAKE",
    "confidence": "high",
    "rationale": "First-time food-assistance application in covered region."
  },
  "jurisdictionVerdict": {
    "allowed": true,
    "violations": [],
    "agencyCode": "INTAKE-01",
    "policyVersion": "v1"
  },
  "intakeOutcome": {
    "segment": "INTAKE",
    "determination": "REFERRED_TO_BENEFITS",
    "summaryNote": "Household of three; program type is active in region. Intake eligibility criteria appear met based on available payload. Benefits segment should calculate award amount. — Intake Assessment · automated draft",
    "agencyTag": "intake",
    "completedAt": "2026-06-29T09:00:14Z"
  },
  "benefitsOutcome": null,
  "handoffApproval": {
    "approvedBy": "officer-chen",
    "note": "Reviewed intake determination; proceeding to benefits.",
    "approvedAt": "2026-06-29T09:02:00Z"
  },
  "handoffRejection": null,
  "status": "HANDOFF_EMITTED",
  "createdAt": "2026-06-29T09:00:00Z",
  "closedAt": null
}
```

A case in `JURISDICTION_BLOCKED` has `jurisdictionVerdict.allowed = false`, non-empty `violations`, and no `intakeOutcome` or `benefitsOutcome`. A case in `HANDOFF_REJECTED` has `handoffRejection` populated and no downstream outcome.

### RoutingDecision (returned by `JurisdictionRouter`, embedded in `Case.routing`)

```json
{ "segment": "INTAKE" | "BENEFITS" | "UNROUTABLE",
  "confidence": "high" | "medium" | "low",
  "rationale": "one short sentence" }
```

### JurisdictionVerdict (returned by `JurisdictionGuardrail`, embedded in `Case.jurisdictionVerdict`)

```json
{ "allowed": true | false,
  "violations": ["geographic-region-out-of-scope", ...],
  "agencyCode": "INTAKE-01" | "BENEFITS-01",
  "policyVersion": "v1" }
```

### SegmentOutcome (returned by either agency agent, embedded in `Case.intakeOutcome` or `Case.benefitsOutcome`)

```json
{ "segment": "INTAKE" | "BENEFITS",
  "determination": "ELIGIBLE" | "INELIGIBLE" | "REFERRED_TO_BENEFITS" | "AWARD_ISSUED" | "AWARD_DENIED" | "PENDING_DOCUMENTS" | "ESCALATED",
  "summaryNote": "2–4 sentences",
  "agencyTag": "intake" | "benefits",
  "completedAt": "ISO-8601" }
```

### HandoffApproval (embedded in `Case.handoffApproval`)

```json
{ "approvedBy": "officer-id",
  "note": "free text",
  "approvedAt": "ISO-8601" }
```

## SSE event format

```
event: case-update
data: { "caseId": "cs-…", "status": "HANDOFF_PENDING", … full Case JSON … }
```

One event per state transition on the `CaseView`. Clients reconcile by `caseId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/cases`).
- `404` — unknown `caseId`.
- `409` — `approve-handoff` or `reject-handoff` requested on a case not currently in `HANDOFF_PENDING`.
- `503` — backend unavailable (workflow timed out, recovery in progress).
