# API contract — pitch-builder

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/pitchbooks` | `SubmitPitchbookRequest` | `201 { pitchbookId }` | `PitchbookEndpoint` → `PitchbookEntity` |
| `GET` | `/api/pitchbooks` | — | `200 [ PitchbookRecord... ]` (newest-first) | `PitchbookEndpoint` ← `PitchbookView` |
| `GET` | `/api/pitchbooks/{id}` | — | `200 PitchbookRecord` / `404` | `PitchbookEndpoint` ← `PitchbookView` |
| `GET` | `/api/pitchbooks/sse` | — | `text/event-stream` | `PitchbookEndpoint` ← `PitchbookView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPitchbookRequest (request body)

```json
{
  "target": "Meridian Packaging Group"
}
```

### PitchbookRecord (response body)

```json
{
  "pitchbookId": "pb-7c3a1f...",
  "target": "Meridian Packaging Group",
  "research": {
    "target": "Meridian Packaging Group",
    "items": [
      {
        "source": "SEC EDGAR",
        "url": "https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&company=meridian+packaging",
        "headline": "Meridian Packaging Group files 10-K for FY2025 with $420M revenue.",
        "metadata": "Contact: [REDACTED]",
        "capturedAt": "2026-06-28T09:00:00Z"
      }
    ],
    "redactionCount": 1,
    "researchedAt": "2026-06-28T09:00:10Z"
  },
  "comps": {
    "peers": [
      { "ticker": "PKG", "name": "Packaging Corporation of America", "sector": "Containers & Packaging", "marketCapM": 14200 },
      { "ticker": "SEE", "name": "Sealed Air Corporation", "sector": "Containers & Packaging", "marketCapM": 4800 }
    ],
    "multiples": {
      "evEbitdaLow": 7.5, "evEbitdaHigh": 11.2,
      "evRevenueLow": 1.1, "evRevenueHigh": 1.9,
      "peRatioLow": 14.0, "peRatioHigh": 20.5
    },
    "builtAt": "2026-06-28T09:00:30Z"
  },
  "pitchbook": {
    "cover": {
      "targetName": "Meridian Packaging Group",
      "sector": "Containers & Packaging",
      "preparedBy": "Pitch Builder",
      "preparedAt": "2026-06-28T09:00:55Z"
    },
    "executiveSummary": "Meridian Packaging Group is a mid-market containers company with $420M in FY2025 revenue. Peer-implied valuation at median EV/EBITDA suggests an enterprise value consistent with a $1.8B–$2.2B range. The investment case rests on margin expansion and packaging-mix shift.",
    "sections": [
      {
        "sectionId": "company-overview",
        "heading": "Company Overview",
        "body": "Meridian Packaging Group operates three manufacturing facilities producing flexible and rigid packaging for consumer goods clients. FY2025 revenue of $420M represents 7% year-over-year growth.",
        "citedTickers": [],
        "citedFigures": []
      },
      {
        "sectionId": "comps-analysis",
        "heading": "Comparable Company Analysis",
        "body": "Meridian's closest public comparables — PKG and SEE — trade at 7.5x–11.2x EV/EBITDA. Applying the midpoint of 9.4x to Meridian's estimated EBITDA of $210M implies an enterprise value of approximately $1.97B.",
        "citedTickers": ["PKG", "SEE"],
        "citedFigures": ["7.5x EV/EBITDA", "11.2x EV/EBITDA", "9.4x EV/EBITDA"]
      }
    ],
    "draftedAt": "2026-06-28T09:01:00Z"
  },
  "validation": {
    "score": 5,
    "rationale": "Ticker coverage, figure provenance, peer name coverage, and section count all satisfied.",
    "validatedAt": "2026-06-28T09:01:01Z"
  },
  "status": "VALIDATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:01:01Z",
  "piiRedactions": [
    {
      "field": "metadata",
      "patternId": "personal-email",
      "offsetStart": 9,
      "offsetEnd": 42,
      "loggedAt": "2026-06-28T09:00:12Z"
    }
  ],
  "citationRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `piiRedactions` is an array — populated when the sanitizer fired on at least one research item field. `citationRejections` is an array — empty on the happy path, populated when the agent cited an unrecorded peer or out-of-range multiple.

### SSE event format

```
event: pitchbook-update
data: { "pitchbookId": "pb-7c3a1f...", "status": "COMPS_READY", "comps": { ... }, ... }
```

One event per state transition (`CREATED`, `RESEARCHING`, `RESEARCHED`, `COMPS_RUNNING`, `COMPS_READY`, `DRAFTING`, `DRAFTED`, `VALIDATED`, `FAILED`) and one each per audit side-event:

```
event: pitchbook-pii-redaction
data: { "pitchbookId": "pb-7c3a1f...", "field": "metadata", "patternId": "personal-email", "loggedAt": "..." }

event: pitchbook-citation-rejection
data: { "pitchbookId": "pb-7c3a1f...", "sectionId": "comps-analysis", "rule": "unknown-ticker", "detail": "ticker XYZ not found in CompsTable.peers", "loggedAt": "..." }
```

Clients reconcile by `pitchbookId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitPitchbookRequest` record and the `PitchbookCreated` event to carry it.
