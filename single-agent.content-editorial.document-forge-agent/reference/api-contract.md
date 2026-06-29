# API contract — document-forge-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/forges` | `ForgeSubmitRequest` | `201 { forgeId }` | `ForgeEndpoint` → `ForgeEntity` |
| `GET` | `/api/forges` | — | `200 [ ForgeRun... ]` (newest-first) | `ForgeEndpoint` ← `ForgeView` |
| `GET` | `/api/forges/{id}` | — | `200 ForgeRun` / `404` | `ForgeEndpoint` ← `ForgeView` |
| `GET` | `/api/forges/sse` | — | `text/event-stream` | `ForgeEndpoint` ← `ForgeView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### ForgeSubmitRequest (request body)

```json
{
  "prompt": "Draft a Q3 roadmap proposal for the infrastructure team, covering three key initiatives and a rough timeline through end of year.",
  "documentType": "PROJECT_PROPOSAL",
  "styleTemplate": "EXECUTIVE_SUMMARY",
  "requestedBy": "author-47"
}
```

### ForgeRun (response body)

```json
{
  "forgeId": "f-7c3...",
  "request": {
    "forgeId": "f-7c3...",
    "prompt": "Draft a Q3 roadmap proposal for the infrastructure team...",
    "documentType": "PROJECT_PROPOSAL",
    "styleTemplate": "EXECUTIVE_SUMMARY",
    "requestedBy": "author-47",
    "submittedAt": "2026-06-28T14:22:00Z"
  },
  "output": {
    "filename": "project-proposal-2026-06.md",
    "content": "## Infrastructure Q3 Roadmap\n\n**Initiative 1: Observability Uplift** ...",
    "formatHint": "markdown",
    "wordCount": 412
  },
  "audit": {
    "score": 4,
    "assessment": "Document meets word count and structure requirements; one section uses a placeholder phrase that should be replaced before distribution.",
    "flaggedPatterns": [],
    "auditedAt": "2026-06-28T14:22:31Z"
  },
  "status": "AUDITED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:31Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: forge-update
data: { "forgeId": "f-7c3...", "status": "FORGE_COMPLETED", "output": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `FORGING`, `FORGE_COMPLETED`, `AUDITED`, `FAILED`). Clients reconcile by `forgeId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
