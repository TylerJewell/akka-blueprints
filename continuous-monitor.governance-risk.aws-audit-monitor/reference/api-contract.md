# API contract — aws-audit-monitor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/audit/findings` | — | `200 [ AuditFinding... ]` (sorted newest-first) | `AuditEndpoint` ← `AuditView` |
| `GET` | `/api/audit/findings/{id}` | — | `200 AuditFinding` / `404` | `AuditEndpoint` ← `AuditView` |
| `GET` | `/api/audit/reports` | — | `200 [ AuditReport... ]` (sorted newest-first) | `AuditEndpoint` ← `AuditView` |
| `GET` | `/api/audit/reports/{id}` | — | `200 AuditReport` / `404` | `AuditEndpoint` ← `AuditView` |
| `POST` | `/api/audit/reports/{id}/publish` | `{ "reviewedBy": String }` | `200 AuditReport` (status now PUBLISHED) | `AuditEndpoint` → `AuditReportEntity` + `AuditFindingEntity` |
| `POST` | `/api/audit/findings/{id}/dismiss` | `{ "reviewedBy": String, "reason": String }` | `200 AuditFinding` (status now DISMISSED) | `AuditEndpoint` → `AuditFindingEntity` |
| `GET` | `/api/audit/sse` | — | `text/event-stream` | `AuditEndpoint` ← `AuditView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### AuditFinding

```json
{
  "findingId": "awsf-3c9a…",
  "raw": {
    "findingId": "awsf-3c9a…",
    "resourceType": "AWS::S3::Bucket",
    "resourceArn": "arn:aws:s3:::my-bucket",
    "ruleId": "s3-bucket-public-read-access",
    "ruleName": "S3 Bucket Public Read Access",
    "rawDetail": "Bucket ACL grants public READ access.",
    "detectedAt": "2026-06-28T09:00:00Z"
  },
  "normalized": {
    "resourceType": "AWS::S3::Bucket",
    "resourceArn": "arn:aws:s3:::my-bucket",
    "ruleId": "s3-bucket-public-read-access",
    "controlDomain": "data-protection",
    "normalizedDetail": "Bucket ACL grants public READ access.",
    "affectedAttributes": ["BucketAcl", "PublicAccessBlock"]
  },
  "analysis": {
    "severity": "CRITICAL",
    "controlRef": "CIS-2.1.5",
    "remediationSummary": "Enable S3 Block Public Access at the account level. Remove the public READ grant from the bucket ACL. Audit object-level ACLs for overrides.",
    "confidence": "high"
  },
  "reportId": "rpt-7f2b…",
  "decision": null,
  "evalScore": null,
  "evalRationale": null,
  "status": "PENDING_REVIEW",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": null
}
```

### AuditReport

```json
{
  "reportId": "rpt-7f2b…",
  "compiled": {
    "reportId": "rpt-7f2b…",
    "executiveSummary": "This scan window produced 5 findings: 1 CRITICAL, 2 HIGH, 1 MEDIUM, 1 LOW. The most urgent control domains are data-protection (S3 public access) and identity (IAM wildcard policy). Recommended priority: remediate the public S3 bucket and IAM wildcard first; the remaining findings can be scheduled for the next sprint.",
    "findingIds": ["awsf-3c9a…", "awsf-1d4e…", "awsf-8b7c…", "awsf-2f1a…", "awsf-6e3d…"],
    "compiledAt": "2026-06-28T09:01:05Z"
  },
  "decision": null,
  "status": "PENDING_REVIEW",
  "createdAt": "2026-06-28T09:00:00Z",
  "publishedAt": null
}
```

### SSE event format

```
event: audit-update
data: { "findingId": "awsf-3c9a…", "status": "PENDING_REVIEW", "severity": "CRITICAL", ... }
```

```
event: report-update
data: { "reportId": "rpt-7f2b…", "status": "PUBLISHED", ... }
```

One event per state transition. Clients reconcile findings by `findingId` and reports by `reportId`.
