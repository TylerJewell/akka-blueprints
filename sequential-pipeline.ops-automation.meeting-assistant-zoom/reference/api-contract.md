# API contract — meeting-assistant-zoom

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/meetings` | `SubmitMeetingRequest` | `201 { meetingId }` | `MeetingEndpoint` → `MeetingEntity` |
| `GET` | `/api/meetings` | — | `200 [ MeetingRecord... ]` (newest-first) | `MeetingEndpoint` ← `MeetingView` |
| `GET` | `/api/meetings/{id}` | — | `200 MeetingRecord` / `404` | `MeetingEndpoint` ← `MeetingView` |
| `GET` | `/api/meetings/sse` | — | `text/event-stream` | `MeetingEndpoint` ← `MeetingView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitMeetingRequest (request body)

```json
{
  "meetingTitle": "Q2 Engineering Retrospective",
  "rawTranscript": "[PERSON-1]: Let's kick off the retro. [PERSON-2]: I'll note the action items..."
}
```

Note: `rawTranscript` is accepted verbatim; the service runs `PiiSanitizer` before any LLM call. The caller does not need to pre-redact the text.

### MeetingRecord (response body)

```json
{
  "meetingId": "m-7bc3e1...",
  "meetingTitle": "Q2 Engineering Retrospective",
  "transcript": {
    "segments": [
      {
        "speaker": "[PERSON-1]",
        "text": "Let's kick off the retro.",
        "startTime": "2026-06-30T14:00:00Z",
        "endTime": "2026-06-30T14:00:04Z"
      }
    ],
    "redactionAudit": {
      "totalRedactions": 4,
      "entries": [
        { "original": "Alice Chen", "pseudonym": "[PERSON-1]", "patternType": "name" },
        { "original": "bob@example.com", "pseudonym": "[EMAIL-1]", "patternType": "email" }
      ]
    },
    "capturedAt": "2026-06-30T14:05:00Z"
  },
  "summary": {
    "summaryText": "The Q2 retro identified three process improvements and confirmed ownership for the upcoming deploy.",
    "actionItems": [
      {
        "actionId": "ai-3f2a1b00",
        "description": "Handle the deploy by Friday",
        "assigneePseudonym": "[PERSON-2]",
        "dueDate": "2026-07-04"
      }
    ],
    "decisions": [
      { "decisionId": "d-1a2b3c4d", "text": "[PERSON-2] owns the deploy", "decidedAt": "2026-06-30T14:02:00Z" }
    ],
    "summarizedAt": "2026-06-30T14:05:10Z"
  },
  "meetingPackage": {
    "title": "Q2 Engineering Retrospective — Follow-ups",
    "summaryText": "The Q2 retro identified three process improvements and confirmed ownership for the upcoming deploy.",
    "actionItems": [
      {
        "actionId": "ai-3f2a1b00",
        "description": "Handle the deploy by Friday",
        "assigneePseudonym": "[PERSON-2]",
        "dueDate": "2026-07-04"
      }
    ],
    "followUpEvents": [
      {
        "eventId": "ev-9d4f2c...",
        "title": "Deploy check-in",
        "attendeePseudonyms": ["[PERSON-1]", "[PERSON-2]"],
        "scheduledDate": "2026-07-03"
      }
    ],
    "tasks": [
      {
        "taskId": "tk-a1b2c3...",
        "assigneePseudonym": "[PERSON-2]",
        "description": "Handle the deploy by Friday",
        "dueDate": "2026-07-04"
      }
    ],
    "packagedAt": "2026-06-30T14:05:20Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Action-item coverage, assignee traceability, event-attendee traceability, and item parity all satisfied.",
    "evaluatedAt": "2026-06-30T14:05:21Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-30T14:00:00Z",
  "finishedAt": "2026-06-30T14:05:21Z",
  "scopeRejections": [
    {
      "phase": "DISPATCH",
      "tool": "createTask",
      "reason": "scope-violation: [PERSON-99] is not a meeting participant",
      "rejectedAt": "2026-06-30T14:05:15Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `scopeRejections` is an array — empty on the happy path, populated when the agent attempted a misordered or out-of-scope tool call.

### SSE event format

```
event: meeting-update
data: { "meetingId": "m-7bc3e1...", "status": "SUMMARIZED", "summary": { ... }, ... }
```

One event per state transition (`CREATED`, `TRANSCRIBING`, `TRANSCRIBED`, `SUMMARIZING`, `SUMMARIZED`, `DISPATCHING`, `DISPATCHED`, `EVALUATED`, `FAILED`) and one per `ScopeRejected` audit event:

```
event: meeting-rejection
data: { "meetingId": "m-7bc3e1...", "phase": "DISPATCH", "tool": "createTask", "reason": "scope-violation: ...", "rejectedAt": "..." }
```

Clients reconcile by `meetingId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitMeetingRequest` record and the `MeetingCreated` event to carry it. Because the raw transcript may contain PII, a deployer in a regulated environment should also add transport-layer encryption (TLS) and at-rest encryption (for the `rawTranscriptEncrypted` field already stored on the entity).
