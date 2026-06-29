# API contract — pokemon-advisor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/advisories` | `SubmitRosterRequest` | `201 { advisoryId }` | `AdvisoryEndpoint` → `RosterEntity` |
| `GET` | `/api/advisories` | — | `200 [ Advisory... ]` (newest-first) | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/advisories/{id}` | — | `200 Advisory` / `404` | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/advisories/sse` | — | `text/event-stream` | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRosterRequest (request body)

```json
{
  "trainerName": "Red",
  "format": "VGC",
  "slots": [
    { "slotNumber": 1, "species": "Incineroar", "nickname": "Brawler" },
    { "slotNumber": 2, "species": "Miraidon",   "nickname": "" },
    { "slotNumber": 3, "species": "Flutter Mane", "nickname": "" },
    { "slotNumber": 4, "species": "Urshifu",    "nickname": "Strike" },
    { "slotNumber": 5, "species": "Rillaboom",  "nickname": "" },
    { "slotNumber": 6, "species": "Amoonguss",  "nickname": "" }
  ]
}
```

### Advisory (response body)

```json
{
  "advisoryId": "a-7f3...",
  "submission": {
    "advisoryId": "a-7f3...",
    "trainerName": "Red",
    "slots": [
      { "slotNumber": 1, "species": "Incineroar", "nickname": "Brawler" },
      { "slotNumber": 2, "species": "Miraidon",   "nickname": "" }
    ],
    "format": "VGC",
    "submittedAt": "2026-06-28T12:34:00Z"
  },
  "validated": {
    "slots": [
      { "slotNumber": 1, "species": "Incineroar", "nickname": "Brawler" },
      { "slotNumber": 2, "species": "Miraidon",   "nickname": "" }
    ],
    "format": "VGC",
    "warningsFound": []
  },
  "recommendation": {
    "verdict": "STRONG",
    "summary": "This VGC core is well-balanced with strong offensive and support roles. Ground-type coverage is a notable gap.",
    "slots": [
      {
        "slotNumber": 1,
        "species": "Incineroar",
        "role": "SUPPORT",
        "suggestedMoves": ["Fake Out", "Flare Blitz", "Parting Shot", "Knock Off"],
        "rationale": "Incineroar's Fake Out and Parting Shot give the team crucial speed control and pivot support."
      },
      {
        "slotNumber": 2,
        "species": "Miraidon",
        "role": "SPECIAL_SWEEPER",
        "suggestedMoves": ["Electro Drift", "Draco Meteor", "Volt Switch", "Protect"],
        "rationale": "Miraidon's exceptional Special Attack and Electric Surge ability anchor the team's offensive core."
      }
    ],
    "gaps": [
      {
        "type": "Ground",
        "severity": "MODERATE",
        "suggestion": "Add Earthquake to Incineroar's move set or replace one slot with a Ground-type Pokemon to cover the Electric weakness."
      }
    ],
    "decidedAt": "2026-06-28T12:34:18Z"
  },
  "coverageScore": {
    "score": 4,
    "rationale": "Team covers five of six primary types; Ground coverage gap reduces score from 5.",
    "scoredAt": "2026-06-28T12:34:19Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: advisory-update
data: { "advisoryId": "a-7f3...", "status": "RECOMMENDATION_RECORDED", "recommendation": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `VALIDATED`, `ADVISING`, `RECOMMENDATION_RECORDED`, `SCORED`, `FAILED`). Clients reconcile by `advisoryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `trainerName` from the authenticated principal rather than the request body.
