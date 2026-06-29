# API contract — medical-document-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/documents` | `SubmitDocumentRequest` | `201 { documentId }` | `DocumentEndpoint` → `DocumentEntity` |
| `GET` | `/api/documents` | — | `200 [ DocumentRecord... ]` (newest-first) | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/{id}` | — | `200 DocumentRecord` / `404` | `DocumentEndpoint` ← `DocumentView` |
| `GET` | `/api/documents/sse` | — | `text/event-stream` | `DocumentEndpoint` ← `DocumentView` |
| `POST` | `/api/documents/{id}/review` | `ClinicalReviewDecision` | `204` | `DocumentEndpoint` → `DocumentEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitDocumentRequest (request body)

```json
{
  "documentType": "discharge-summary",
  "rawText": "Patient [raw text with PHI — masked before any agent call]..."
}
```

`documentType` values: `"discharge-summary"` | `"radiology-report"` | `"referral-letter"`.

### ClinicalReviewDecision (request body for `/review`)

```json
{
  "approvedFields": [
    {
      "fieldId": "f-e5f6a7b8",
      "fieldName": "primary_diagnosis",
      "extractedValue": "Acute STEMI, anterior wall",
      "confidence": 0.94,
      "sourceSpan": "88:120"
    }
  ],
  "corrections": [
    {
      "fieldId": "f-c9d0e1f2",
      "correctedValue": "Type 2 diabetes mellitus, diet-controlled"
    }
  ],
  "reviewedBy": "[CLINICIAN_TOKEN]"
}
```

`corrections` may be an empty array. For each entry in `corrections`, the server applies the corrected value to the matching `approvedFields` entry before writing `ReviewSubmitted`. `reviewedBy` is a clinician identifier token — the deployer is responsible for mapping this to an authenticated session.

### DocumentRecord (response body)

```json
{
  "documentId": "doc-7b3e91...",
  "documentType": "discharge-summary",
  "maskedDocument": {
    "documentId": "doc-7b3e91...",
    "documentType": "discharge-summary",
    "maskedText": "Patient [PATIENT_NAME], DOB [DOB], MRN [MRN], admitted with...",
    "tokens": [
      { "token": "[PATIENT_NAME]", "category": "PATIENT_NAME" },
      { "token": "[DOB]", "category": "DOB" },
      { "token": "[MRN]", "category": "MRN" }
    ],
    "maskedAt": "2026-06-28T10:00:02Z"
  },
  "extractedFields": {
    "demographics": [
      { "fieldId": "f-a1b2c3d4", "fieldName": "patient_age", "extractedValue": "67", "confidence": 0.97, "sourceSpan": "12:14" }
    ],
    "diagnoses": [
      { "fieldId": "f-e5f6a7b8", "fieldName": "primary_diagnosis", "extractedValue": "Acute STEMI, anterior wall", "confidence": 0.94, "sourceSpan": "88:120" }
    ],
    "medications": [
      { "fieldId": "f-a3b4c5d6", "fieldName": "medication", "extractedValue": "Aspirin 81 mg daily", "confidence": 0.99, "sourceSpan": "203:222" }
    ],
    "vitals": [
      { "fieldId": "f-e7f8a9b0", "fieldName": "heart_rate", "extractedValue": "88 bpm", "confidence": 0.98, "sourceSpan": "301:307" }
    ],
    "extractedAt": "2026-06-28T10:00:15Z"
  },
  "validationResult": {
    "approvedFields": [
      { "fieldId": "f-e5f6a7b8", "fieldName": "primary_diagnosis", "extractedValue": "Acute STEMI, anterior wall", "confidence": 0.94, "sourceSpan": "88:120" }
    ],
    "findings": [
      { "findingId": "vf-001", "fieldId": "f-e7f8a9b0", "severity": "OK", "message": "Heart rate 88 bpm within normal range." }
    ],
    "reviewedBy": "[CLINICIAN_TOKEN]",
    "reviewedAt": "2026-06-28T10:02:30Z"
  },
  "clinicalSummary": {
    "documentType": "discharge-summary",
    "patientContext": "67-year-old patient ([PATIENT_NAME]) discharged following cardiac intervention.",
    "chiefComplaint": "Acute anterior STEMI presenting with chest pain and ST-elevation on ECG.",
    "sections": [
      {
        "sectionId": "sec-01",
        "heading": "Primary Diagnosis",
        "body": "Patient diagnosed with acute anterior STEMI. Cardiac catheterisation performed; culprit lesion identified and treated.",
        "sourceFieldIds": ["f-e5f6a7b8"]
      }
    ],
    "clinicalImpression": "Haemodynamically stable at discharge with optimised antiplatelet and statin therapy initiated.",
    "recommendations": "Follow up with cardiology in 4 weeks. Referral to cardiac rehabilitation programme.",
    "writtenAt": "2026-06-28T10:03:45Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Mandatory field coverage, value provenance, and section completeness all satisfied; one section body marginally short.",
    "evaluatedAt": "2026-06-28T10:03:46Z"
  },
  "status": "EVALUATED",
  "sanitized": true,
  "receivedAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:03:46Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path.

A document in `AWAITING_REVIEW` has `extractedFields` populated and `validationResult: null`; the UI renders the clinician review form when this combination is detected.

### SSE event format

```
event: document-update
data: { "documentId": "doc-7b3e91...", "status": "AWAITING_REVIEW", "extractedFields": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZING`, `SANITIZED`, `EXTRACTING`, `EXTRACTED`, `AWAITING_REVIEW`, `REVIEW_SUBMITTED`, `SUMMARIZING`, `SUMMARIZED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: document-rejection
data: { "documentId": "doc-7b3e91...", "phase": "EXTRACT", "tool": "buildClinicalImpression", "reason": "phase-violation: buildClinicalImpression requires status in {REVIEW_SUBMITTED, SUMMARIZING} with validationResult present, saw EXTRACTING", "rejectedAt": "2026-06-28T10:00:05Z" }
```

Clients reconcile by `documentId`; every event carries the full row at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp `submittedBy` and `reviewedBy` fields from the authenticated principal — extend `SubmitDocumentRequest`, `ClinicalReviewDecision`, and the corresponding events accordingly. The `reviewedBy` field in `ClinicalReviewDecision` must map to a credentialed clinician in the deployer's identity provider.
