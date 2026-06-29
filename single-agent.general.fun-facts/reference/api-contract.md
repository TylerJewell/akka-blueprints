# API contract — fun-facts

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/fact-requests` | `SubmitFactRequest` | `201 { requestId }` | `FactEndpoint` → `FactRequestEntity` |
| `GET` | `/api/fact-requests` | — | `200 [ FactRequestRow... ]` (newest-first) | `FactEndpoint` ← `FactRequestView` |
| `GET` | `/api/fact-requests/{id}` | — | `200 FactRequestRow` / `404` | `FactEndpoint` ← `FactRequestView` |
| `GET` | `/api/fact-requests/sse` | — | `text/event-stream` | `FactEndpoint` ← `FactRequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitFactRequest (request body)

```json
{
  "topic": "Octopus Intelligence",
  "requestedBy": "curious-user-42"
}
```

Validation: `topic` must be a non-blank string. If blank, the endpoint returns `400` with body:

```json
{ "error": "topic must not be blank" }
```

No entity is created on validation failure.

### FactRequestRow (response body)

```json
{
  "requestId": "fr-9a3b...",
  "request": {
    "requestId": "fr-9a3b...",
    "topic": "Octopus Intelligence",
    "requestedBy": "curious-user-42",
    "requestedAt": "2026-06-28T12:00:00Z"
  },
  "collection": {
    "headlineFact": "Octopuses have three hearts: two pump blood to the gills and one pumps it to the rest of the body — and the systemic heart stops beating while the octopus swims, which is why they prefer crawling.",
    "facts": [
      {
        "area": "cognition",
        "statement": "Octopuses can unscrew jar lids and navigate mazes, demonstrating problem-solving that relies on distributed intelligence: two-thirds of their neurons are located in their arms rather than their central brain.",
        "confidence": "HIGH"
      },
      {
        "area": "vision",
        "statement": "Octopuses are colourblind yet produce elaborate colour-changing displays; researchers hypothesise they may perceive colour through photoreceptors distributed across their skin.",
        "confidence": "MEDIUM"
      },
      {
        "area": "lifespan",
        "statement": "Most octopus species live 1–2 years; the giant Pacific octopus lives up to 5 years, making it unusually long-lived for a cephalopod.",
        "confidence": "HIGH"
      }
    ],
    "generatedAt": "2026-06-28T12:00:08Z"
  },
  "status": "GENERATED",
  "createdAt": "2026-06-28T12:00:00Z",
  "finishedAt": "2026-06-28T12:00:08Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: fact-request-update
data: { "requestId": "fr-9a3b...", "status": "GENERATED", "collection": { ... }, ... }
```

One event per state transition (`PENDING`, `GENERATING`, `GENERATED`, `FAILED`). Clients reconcile by `requestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
