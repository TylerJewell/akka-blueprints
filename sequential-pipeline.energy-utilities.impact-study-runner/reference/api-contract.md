# API contract — impact-study-runner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/studies` | `SubmitStudyRequest` | `201 { studyId }` | `StudyEndpoint` → `StudyEntity` |
| `GET` | `/api/studies` | — | `200 [ StudyRecord... ]` (newest-first) | `StudyEndpoint` ← `StudyView` |
| `GET` | `/api/studies/{id}` | — | `200 StudyRecord` / `404` | `StudyEndpoint` ← `StudyView` |
| `GET` | `/api/studies/sse` | — | `text/event-stream` | `StudyEndpoint` ← `StudyView` |
| `POST` | `/api/studies/{id}/approve` | `ApproveRequest` | `200 { studyId, status }` | `StudyEndpoint` → `StudyEntity` |
| `POST` | `/api/studies/{id}/reject` | `RejectRequest` | `200 { studyId, status }` | `StudyEndpoint` → `StudyEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitStudyRequest (request body)

```json
{
  "requestId": "REQ-2026-001 Lakeside Wind 150 MW"
}
```

### ApproveRequest (request body)

```json
{
  "note": "All findings reviewed. Voltage profile acceptable. Recommend conditional approval pending mitigation of LINE-202 overload."
}
```

`note` is optional.

### RejectRequest (request body)

```json
{
  "reason": "N-1 criterion not met. Binding violation on LINE-202 requires mitigation study before re-submission."
}
```

### StudyRecord (response body)

```json
{
  "studyId": "s-7bc4a1...",
  "requestId": "REQ-2026-001 Lakeside Wind 150 MW",
  "loadFlow": {
    "busReadings": [
      { "busId": "BUS-101", "busName": "Lakeside 345kV", "voltageKv": 345.8, "voltagePu": 1.002, "measuredAt": "2026-06-29T10:00:00Z" }
    ],
    "lineFlows": [
      { "lineId": "LINE-201", "fromBus": "BUS-101", "toBus": "BUS-102", "flowMw": 118.4, "loadPercent": 78.9, "measuredAt": "2026-06-29T10:00:00Z" }
    ],
    "computedAt": "2026-06-29T10:00:00Z"
  },
  "contingency": {
    "violations": [
      { "contingencyId": "CONT-LINE-201-OUT", "elementName": "LINE-202", "limitType": "thermal-MW", "limit": 120.0, "observed": 134.7, "isBinding": true }
    ],
    "voltageChecks": [
      { "busId": "BUS-101", "voltageKv": 345.8, "lowerLimitKv": 330.0, "upperLimitKv": 360.0, "withinLimits": true }
    ],
    "n1ContingenciesTested": 4,
    "analyzedAt": "2026-06-29T10:00:05Z"
  },
  "report": {
    "requestId": "REQ-2026-001 Lakeside Wind 150 MW",
    "executiveSummary": "The Lakeside Wind 150 MW interconnection fails the N-1 contingency criterion under the outage of LINE-201.",
    "sections": [
      {
        "findingId": "CONT-LINE-201-OUT",
        "heading": "N-1 overload: LINE-202 under LINE-201 outage",
        "body": "With LINE-201 out of service, LINE-202 carries 134.7 MW against a thermal limit of 120.0 MW.",
        "busRefs": [{ "busId": "BUS-101", "busName": "Lakeside 345kV" }]
      }
    ],
    "meetsN1Criterion": false,
    "draftedAt": "2026-06-29T10:00:10Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Violation coverage, voltage bounds, N-1 criterion flag, and section count parity all satisfied.",
    "evaluatedAt": "2026-06-29T10:00:11Z"
  },
  "approvalNote": null,
  "status": "AWAITING_APPROVAL",
  "createdAt": "2026-06-29T10:00:00Z",
  "finishedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `approvalNote` is `null` until `StudyApproved` lands.

### SSE event format

```
event: study-update
data: { "studyId": "s-7bc4a1...", "status": "AWAITING_APPROVAL", "report": { ... }, "eval": { ... }, ... }
```

One event per state transition (`CREATED`, `RUNNING_LOAD_FLOW`, `LOAD_FLOW_DONE`, `RUNNING_CONTINGENCY`, `CONTINGENCY_DONE`, `DRAFTING`, `REPORT_DRAFTED`, `AWAITING_APPROVAL`, `APPROVED`, `REJECTED`, `FAILED`).

Clients reconcile by `studyId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the approve/reject endpoints with their auth middleware and restrict them to engineers who hold the `STUDY_APPROVER` role — extend `StudyEndpoint` to validate the caller's role claim before routing to `StudyEntity.approve` or `StudyEntity.reject`.
