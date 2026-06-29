# API contract — Meeting Prep Briefer

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Briefing surface (BriefingEndpoint, `/api`)

### POST /api/briefings
Body:
```json
{ "meetingTopic": "string", "participants": ["string", "..."] }
```
Response `200`:
```json
{ "briefingId": "uuid" }
```
Enqueues a request on `RequestQueue`; the consumer starts a `BriefingWorkflow`.

### GET /api/briefings
Optional `?status=RECEIVED|PLANNING|RESEARCHING|COMPOSING|READY|FAILED`. The endpoint queries `BriefingsView.getAllBriefings` and filters by status client-side (the view cannot index the enum column).
Response `200`:
```json
{ "briefings": [ Briefing, "..." ] }
```

### GET /api/briefings/{id}
Response `200` a single `Briefing`, or `404`.

### GET /api/briefings/sse
`text/event-stream`. Emits a `Briefing` JSON object on every entity change.

### Metadata
```
GET /api/metadata/eval-matrix   -> text/yaml
GET /api/metadata/risk-survey   -> text/yaml
GET /api/metadata/readme        -> text/markdown
```
Served from `src/main/resources/metadata/`.

## Research surface (ResearchSource, `/research`)

In-process stand-in for an external research API. The worker agents call it as a tool; the G1 guardrail gates the call.

```
GET /research/participant?name=<name>   -> { "name": "...", "role": "...",
                                             "raw": "...background incl. email/phone/address..." }
GET /research/topic?q=<topic>           -> { "summary": "...", "points": ["...", "..."] }
```
Canned payloads come from `src/main/resources/sample-data/participants.json` and `topics.json`. Participant payloads deliberately include email/phone/address so the S1 sanitizer has PII to redact.

## UI surface (AppEndpoint)

```
GET /                 -> 302 /app/index.html
GET /app/{*path}      -> static-resources/{*path}
```

## Briefing JSON form

Lifecycle fields are nullable on the wire (Java `Optional`):

```json
{
  "id": "uuid",
  "meetingTopic": "string",
  "participants": ["string"],
  "status": "RECEIVED | PLANNING | RESEARCHING | COMPOSING | READY | FAILED",
  "plannedAt": "ISO-8601 or null",
  "researchPlan": ["string"] ,
  "profiles": [ { "name": "string", "role": "string", "redactedBackground": "string" } ],
  "topicBrief": { "summary": "string", "keyPoints": ["string"] },
  "briefingDoc": "markdown string or null",
  "talkingPoints": ["string"],
  "questions": ["string"],
  "readyAt": "ISO-8601 or null",
  "failedAt": "ISO-8601 or null",
  "failureReason": "string or null"
}
```

## SSE event format

Each `/api/briefings/sse` message is a bare `Briefing` JSON object (no envelope), one per entity update. Clients key on `id` and replace the prior copy in their local list.
