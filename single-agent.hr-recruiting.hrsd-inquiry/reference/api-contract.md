# API contract — hrsd-inquiry

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/inquiries` | `SubmitInquiryRequest` | `201 { inquiryId }` | `InquiryEndpoint` → `InquiryEntity` |
| `GET` | `/api/inquiries` | — | `200 [ Inquiry... ]` (newest-first) | `InquiryEndpoint` ← `InquiryView` |
| `GET` | `/api/inquiries/{id}` | — | `200 Inquiry` / `404` | `InquiryEndpoint` ← `InquiryView` |
| `GET` | `/api/inquiries/sse` | — | `text/event-stream` | `InquiryEndpoint` ← `InquiryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitInquiryRequest (request body)

```json
{
  "employeeId": "emp-00412",
  "topic": "LEAVE_AND_ABSENCE",
  "rawMessage": "I was recently diagnosed with a condition and need to take some time off. How does FMLA work and can I request leave now?",
  "submitRequestIfApplicable": true
}
```

### Inquiry (response body)

```json
{
  "inquiryId": "i-7c4...",
  "request": {
    "inquiryId": "i-7c4...",
    "employeeId": "emp-00412",
    "topic": "LEAVE_AND_ABSENCE",
    "rawMessage": "I was recently diagnosed with a condition and need to take some time off. How does FMLA work and can I request leave now?",
    "submitRequestIfApplicable": true,
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "screened": {
    "redactedMessage": "I was recently [REDACTED-HEALTH] and need to take some time off. How does FMLA work and can I request leave now?",
    "specialCategoriesFound": ["health-condition"]
  },
  "response": {
    "answer": "Under policy LEA-203, employees are eligible for up to 12 weeks of unpaid FMLA leave for qualifying medical events. You can initiate a leave request at any time by submitting form HR-42.",
    "citedPolicies": [
      { "policyId": "LEA-203", "title": "FMLA Leave Entitlement", "section": "Section 2: Eligibility" },
      { "policyId": "LEA-210", "title": "Leave Request Process", "section": "Section 1: Submission" }
    ],
    "serviceRequest": {
      "requestType": "LEAVE_REQUEST",
      "description": "Employee requests initiation of FMLA leave process",
      "fields": { "leaveType": "FMLA", "formRef": "HR-42" }
    },
    "answeredAt": "2026-06-28T14:00:22Z"
  },
  "serviceRequestRef": "HR-7C4ABCDE-54321",
  "status": "REQUEST_SUBMITTED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: inquiry-update
data: { "inquiryId": "i-7c4...", "status": "ANSWERED", "response": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SCREENED`, `ANSWERING`, `ANSWERED`, `REQUEST_SUBMITTED`, `FAILED`). Clients reconcile by `inquiryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and resolve `employeeId` from the authenticated principal rather than the request body. Employee access to another employee's inquiry should be denied by the auth layer; the service itself does not enforce cross-employee isolation.
