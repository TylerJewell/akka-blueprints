# API contract — financial-research-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/reports` | `{ "topic": String, "sectorTag"?: String, "wordCeiling"?: Integer, "requestedBy"?: String }` | `202 { "reportId": String }` | `ResearchEndpoint` → `QueryQueue` |
| `GET` | `/api/reports` | — | `200 [ Report... ]` (optional `?status=PLANNING\|RESEARCHING\|ANALYSING\|WRITING\|VERIFYING\|AWAITING_COMPLIANCE\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllReports`) | `ResearchEndpoint` ← `ReportsView` |
| `GET` | `/api/reports/{id}` | — | `200 Report` / `404` | `ResearchEndpoint` ← `ReportsView` |
| `GET` | `/api/reports/sse` | — | `text/event-stream` (one event per report change) | `ResearchEndpoint` ← `ReportsView` |
| `POST` | `/api/reports/{id}/approve-compliance` | `{ "reviewerId"?: String, "notes"?: String }` | `200 { "reportId": String, "status": "APPROVED" }` / `404` / `409` (wrong state) | `ResearchEndpoint` → `ResearchEntity` |
| `POST` | `/api/reports/{id}/reject-compliance` | `{ "reviewerId"?: String, "reason": String }` | `200 { "reportId": String, "status": "REJECTED_FINAL" }` / `404` / `409` | `ResearchEndpoint` → `ResearchEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/reports`

- Missing `sectorTag` → `"general"`.
- Missing `wordCeiling` → `800` (from `financial-research-pipeline.research.default-word-ceiling`).
- Missing `requestedBy` → `"anonymous"`.
- `wordCeiling` must be in `[200, 5000]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(topic, requestedBy)`; the second submission returns the first `reportId` (200) instead of starting a new workflow.

## JSON shapes

### Report

```json
{
  "reportId": "r-4a8c1…",
  "topic": "Compare the free cash flow profiles of the three largest US semiconductor companies",
  "sectorTag": "equity",
  "wordCeiling": 800,
  "maxVerifyAttempts": 3,
  "status": "APPROVED",
  "plan": {
    "subTasks": [
      { "taskNumber": 1, "scope": "Retrieve FCF figures for NVIDIA FY2025", "sectorTag": "equity" },
      { "taskNumber": 2, "scope": "Retrieve FCF figures for Intel FY2025", "sectorTag": "equity" },
      { "taskNumber": 3, "scope": "Retrieve FCF figures for Qualcomm FY2025", "sectorTag": "equity" }
    ],
    "planSummary": "Three parallel retrieval tasks targeting FCF series per company. Ordered for cross-company consistency.",
    "plannedAt": "2026-06-28T09:00:01Z"
  },
  "sourceBundles": [
    {
      "taskNumber": 1,
      "excerpts": [
        { "text": "NVIDIA reported FCF of $43.2B in FY2025.", "provenance": "NVIDIA FY2025 Annual Report, p. 42", "relevanceScore": 0.97 }
      ],
      "retrievedAt": "2026-06-28T09:00:15Z"
    }
  ],
  "analysisSections": [
    {
      "sectionNumber": 1,
      "heading": "FCF scale and growth",
      "claims": ["NVIDIA generated $43.2B in FCF in FY2025, up 47% YoY."],
      "citations": ["NVIDIA FY2025 Annual Report, p. 42"]
    }
  ],
  "sanitizerLog": { "changed": false, "removedItems": [], "reasonCode": "SECTOR_SCRUB" },
  "drafts": [
    {
      "draftNumber": 1,
      "text": "NVIDIA generated $43.2B in free cash flow in FY2025…",
      "wordCount": 820,
      "draftedAt": "2026-06-28T09:01:10Z"
    },
    {
      "draftNumber": 2,
      "text": "NVIDIA generated $43.2B in free cash flow in FY2025…",
      "wordCount": 780,
      "draftedAt": "2026-06-28T09:01:30Z"
    }
  ],
  "verifications": [
    {
      "verdict": "REVISE",
      "notes": {
        "bullets": ["Paragraph 2 omits the strategic-intent context for Intel's capex."],
        "overallRationale": "Source coverage falls below threshold for Intel section."
      },
      "score": 3,
      "verifiedAt": "2026-06-28T09:01:45Z"
    },
    {
      "verdict": "APPROVE",
      "notes": { "bullets": [], "overallRationale": "All claims grounded; coverage complete; register appropriate." },
      "score": 5,
      "verifiedAt": "2026-06-28T09:02:10Z"
    }
  ],
  "approvedDraftNumber": 3,
  "approvedText": "NVIDIA generated $43.2B in free cash flow…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:03:01Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",                "detail": "" }
{ "passed": false, "reasonCode": "OVER_WORD_CEILING",  "detail": "Draft is 820 words; ceiling is 800." }
```

`reasonCode` is one of `OK`, `OVER_WORD_CEILING`. Additional codes can be added without breaking the contract.

### SSE event format

```
event: report-update
data: { "reportId": "r-4a8c1…", "status": "VERIFYING", "drafts": [...], "verifications": [...], ... }
```

One event per state transition. Clients reconcile by `reportId`. The full `Report` JSON is included so a fresh client can render the row without a separate fetch.

### Compliance action request

```json
{ "reviewerId": "compliance-officer-01", "notes": "Review complete; no MNPI identified." }
```

`reviewerId` defaults to `"anonymous-reviewer"` if omitted.
