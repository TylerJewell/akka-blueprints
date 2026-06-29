# API contract — earth-engine-geospatial

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/analyses` | `SubmitAnalysisRequest` | `201 { analysisId }` | `AnalysisEndpoint` → `AnalysisEntity` |
| `GET` | `/api/analyses` | — | `200 [ Analysis... ]` (newest-first) | `AnalysisEndpoint` ← `AnalysisView` |
| `GET` | `/api/analyses/{id}` | — | `200 Analysis` / `404` | `AnalysisEndpoint` ← `AnalysisView` |
| `GET` | `/api/analyses/sse` | — | `text/event-stream` | `AnalysisEndpoint` ← `AnalysisView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAnalysisRequest (request body)

```json
{
  "queryLabel": "Amazon deforestation front — Q3 2024",
  "bounds": {
    "swLat": -10.5,
    "swLon": -65.0,
    "neLat": -6.0,
    "neLon": -58.0
  },
  "startDate": "2024-07-01",
  "endDate": "2024-09-30",
  "indicators": [
    {
      "indicatorId": "veg-ndvi",
      "name": "Normalized Difference Vegetation Index",
      "datasetLayer": "MODIS/006/MOD13A2",
      "unit": "dimensionless"
    },
    {
      "indicatorId": "veg-evi",
      "name": "Enhanced Vegetation Index",
      "datasetLayer": "MODIS/006/MOD13A2",
      "unit": "dimensionless"
    }
  ],
  "submittedBy": "researcher-88"
}
```

### Analysis (response body)

```json
{
  "analysisId": "a-3fc...",
  "query": {
    "analysisId": "a-3fc...",
    "queryLabel": "Amazon deforestation front — Q3 2024",
    "bounds": { "swLat": -10.5, "swLon": -65.0, "neLat": -6.0, "neLon": -58.0 },
    "startDate": "2024-07-01",
    "endDate": "2024-09-30",
    "indicators": [
      { "indicatorId": "veg-ndvi", "name": "Normalized Difference Vegetation Index",
        "datasetLayer": "MODIS/006/MOD13A2", "unit": "dimensionless" }
    ],
    "submittedBy": "researcher-88",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "dataset": {
    "layers": [
      {
        "indicatorId": "veg-ndvi",
        "datasetLayer": "MODIS/006/MOD13A2",
        "meanValue": 0.19,
        "minValue": 0.04,
        "maxValue": 0.51,
        "missingDataPct": 3.2,
        "unit": "dimensionless"
      }
    ],
    "pixelResolutionMeters": 500,
    "totalPixels": 4820000
  },
  "report": {
    "status": "ANOMALY_DETECTED",
    "summary": "NDVI is well below the Q3 seasonal baseline for this region. The magnitude of the deviation is consistent with active deforestation or severe drought stress.",
    "findings": [
      {
        "indicatorId": "veg-ndvi",
        "severity": "ALERT",
        "datasetLayer": "MODIS/006/MOD13A2",
        "statistic": "mean NDVI = 0.19 over 2024-07 to 2024-09 (seasonal baseline 0.42)",
        "anomalyDetected": true,
        "anomalyConfidence": 0.87,
        "recommendation": "Cross-check against SAR backscatter for the same period to distinguish canopy loss from drought-induced senescence."
      }
    ],
    "decidedAt": "2026-06-28T09:00:28Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Every finding cites a layer and a quantitative statistic; recommendations are specific and actionable.",
    "evaluatedAt": "2026-06-28T09:00:29Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:29Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: analysis-update
data: { "analysisId": "a-3fc...", "status": "REPORT_RECORDED", "report": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `DATASET_READY`, `ANALYSING`, `REPORT_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `analysisId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
