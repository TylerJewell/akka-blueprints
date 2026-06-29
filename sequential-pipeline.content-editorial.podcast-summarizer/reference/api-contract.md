# API contract — podcast-summarizer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/episodes` | `SubmitEpisodeRequest` | `201 { episodeId }` | `EpisodeEndpoint` → `EpisodeEntity` |
| `GET` | `/api/episodes` | — | `200 [ EpisodeRecord... ]` (newest-first) | `EpisodeEndpoint` ← `EpisodeView` |
| `GET` | `/api/episodes/{id}` | — | `200 EpisodeRecord` / `404` | `EpisodeEndpoint` ← `EpisodeView` |
| `GET` | `/api/episodes/sse` | — | `text/event-stream` | `EpisodeEndpoint` ← `EpisodeView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitEpisodeRequest (request body)

```json
{
  "episodeTitle": "Deep Cuts: Postgres internals with Bruce Momjian",
  "transcriptText": "[full transcript text or empty string to use seeded sample data]"
}
```

### EpisodeRecord (response body)

```json
{
  "episodeId": "ep-5af9c2...",
  "episodeTitle": "Deep Cuts: Postgres internals with Bruce Momjian",
  "transcriptText": "...",
  "quotes": {
    "quotes": [
      {
        "quoteId": "q-3a9f11c4",
        "speaker": "Bruce Momjian",
        "text": "MVCC in Postgres means every write is invisible until the transaction commits.",
        "timestampLabel": "08:42"
      }
    ],
    "episodeTitle": "Deep Cuts: Postgres internals with Bruce Momjian",
    "extractedAt": "2026-06-28T10:00:00Z"
  },
  "segmentation": {
    "clusters": [
      {
        "clusterId": "mvcc-isolation",
        "topicLabel": "MVCC and isolation",
        "quoteIds": ["q-3a9f11c4"]
      }
    ],
    "segmentedAt": "2026-06-28T10:00:05Z"
  },
  "summary": {
    "episodeTitle": "Deep Cuts: Postgres internals with Bruce Momjian",
    "overallTakeaway": "Postgres's MVCC model prioritises concurrent read throughput...",
    "sections": [
      {
        "clusterId": "mvcc-isolation",
        "heading": "MVCC and isolation",
        "body": "Bruce Momjian explains that Postgres's MVCC model...",
        "supportingQuoteIds": ["q-3a9f11c4"]
      }
    ],
    "writtenAt": "2026-06-28T10:00:10Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Cluster coverage, quote attribution, quote provenance, and section parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z",
  "guardrailRejections": [
    {
      "phase": "EXTRACT",
      "tool": "draftSection",
      "reason": "phase-violation: draftSection requires status in {SEGMENTED, SUMMARIZING} with segmentation present, saw EXTRACTING",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

### SSE event format

```
event: episode-update
data: { "episodeId": "ep-5af9c2...", "status": "SEGMENTED", "segmentation": { ... }, ... }
```

One event per state transition (`CREATED`, `EXTRACTING`, `EXTRACTED`, `SEGMENTING`, `SEGMENTED`, `SUMMARIZING`, `SUMMARIZED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: episode-rejection
data: { "episodeId": "ep-5af9c2...", "phase": "EXTRACT", "tool": "draftSection", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `episodeId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Owning component per endpoint

| Endpoint | Owning component | Notes |
|---|---|---|
| `POST /api/episodes` | `EpisodeEndpoint` | Mints `episodeId`, creates entity, starts workflow. |
| `GET /api/episodes` | `EpisodeEndpoint` ← `EpisodeView` | Query `getAllEpisodes`; sorted newest-first in the endpoint. |
| `GET /api/episodes/{id}` | `EpisodeEndpoint` ← `EpisodeView` | Single row lookup by `episodeId`. |
| `GET /api/episodes/sse` | `EpisodeEndpoint` ← `EpisodeView` | Forwards the view's `streamUpdates` as Server-Sent Events. |
| `GET /api/metadata/*` | `AppEndpoint` | Reads files from `src/main/resources/metadata/` on classpath. |
| `GET /` | `AppEndpoint` | 302 redirect to `/app/index.html`. |
| `GET /app/*` | `AppEndpoint` | Serves `static-resources/` from classpath. |

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitEpisodeRequest` record and the `EpisodeCreated` event to carry it.
