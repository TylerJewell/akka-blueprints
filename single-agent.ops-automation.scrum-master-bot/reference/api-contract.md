# API contract — scrum-master-bot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/standups` | `StartStandupRequest` | `201 { sessionId }` | `StandupEndpoint` → `StandupEntity` |
| `GET` | `/api/standups` | — | `200 [ StandupSession... ]` (newest-first) | `StandupEndpoint` ← `StandupView` |
| `GET` | `/api/standups/{id}` | — | `200 StandupSession` / `404` | `StandupEndpoint` ← `StandupView` |
| `GET` | `/api/standups/sse` | — | `text/event-stream` | `StandupEndpoint` ← `StandupView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartStandupRequest (request body)

```json
{
  "sprintId": "sp-2026-06-feature-12",
  "teamName": "Platform Feature Team",
  "members": [
    { "memberId": "alice", "displayName": "Alice Chen", "role": "Senior Engineer" },
    { "memberId": "bob",   "displayName": "Bob Ruiz",   "role": "Engineer" },
    { "memberId": "carol", "displayName": "Carol Lee",  "role": "QA Engineer" }
  ],
  "authorizedTicketIds": ["FEAT-10", "FEAT-11", "FEAT-12", "FEAT-13", "FEAT-14", "BUG-42"]
}
```

### StandupSession (response body)

```json
{
  "sessionId": "ss-3a9...",
  "sprintContext": {
    "sprintId": "sp-2026-06-feature-12",
    "sprintName": "Feature Sprint 12",
    "sprintNumber": 12,
    "teamName": "Platform Feature Team",
    "members": [
      { "memberId": "alice", "displayName": "Alice Chen", "role": "Senior Engineer" }
    ],
    "authorizedTicketIds": ["FEAT-10", "FEAT-11", "FEAT-12", "FEAT-13", "FEAT-14", "BUG-42"],
    "sprintStartAt": "2026-06-21T00:00:00Z",
    "sprintEndAt": "2026-07-04T23:59:59Z"
  },
  "summary": {
    "sessionOutcome": "AT_RISK",
    "summaryText": "Alice is progressing on dashboard; Bob is blocked on design sign-off for FEAT-11; Carol is clear. Sprint is at risk if the FEAT-11 blocker persists.",
    "memberUpdates": [
      {
        "memberId": "alice",
        "yesterday": "Completed login flow and unit tests.",
        "today": "Starting dashboard component.",
        "blocker": null
      },
      {
        "memberId": "bob",
        "yesterday": "Reviewed FEAT-11 requirements.",
        "today": "Waiting on design sign-off.",
        "blocker": "Design sign-off on FEAT-11 not received."
      },
      {
        "memberId": "carol",
        "yesterday": "Ran regression suite for FEAT-10.",
        "today": "Writing test cases for dashboard.",
        "blocker": null
      }
    ],
    "ticketUpdates": [
      {
        "ticketId": "FEAT-10",
        "comment": "Standup 2026-06-28: Login flow complete, tests passing. Alice moving to dashboard.",
        "newStatus": null
      },
      {
        "ticketId": "FEAT-11",
        "comment": "Standup 2026-06-28: Blocked on design sign-off. No code progress.",
        "newStatus": null
      }
    ],
    "nextActions": [
      "Follow up with design lead on FEAT-11 sign-off before end of day.",
      "Escalate to PM if sign-off not received by 15:00."
    ],
    "conductedAt": "2026-06-28T09:00:00Z"
  },
  "postResult": {
    "ticketsPosted": 2,
    "skippedTicketIds": [],
    "postedAt": "2026-06-28T09:00:05Z"
  },
  "status": "POSTED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:05Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "ss-3a9...", "status": "SUMMARY_READY", "summary": { ... }, ... }
```

One event per state transition (`COLLECTING`, `RUNNING`, `SUMMARY_READY`, `POSTED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and pass the authenticated team identifier as part of the sprint context rather than relying on the request body's `teamName` field.
