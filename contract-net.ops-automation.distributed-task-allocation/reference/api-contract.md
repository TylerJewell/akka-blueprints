# API Contract — Contract-Net Task Auctioneer

Base URL: `http://localhost:9202`

All request and response bodies are `application/json`. SSE endpoints produce `text/event-stream`.

---

## POST /tasks

Announce a new task and open the bidding window.

**Owning component:** `TaskAuctioneerEndpoint` → `TaskAnnouncementEntity.AnnounceTask`

**Request body**
```json
{
  "title": "string — required",
  "description": "string — required, full requirements text",
  "requiredCapabilities": ["string"],
  "maxBudget": "decimal — required",
  "biddingDeadlineSeconds": "integer — required, minimum 10"
}
```

**Response `201 Created`**
```json
{
  "taskId": "string — UUID assigned by the service",
  "status": "ANNOUNCED",
  "announcedAt": "ISO-8601 timestamp"
}
```

**Errors**
- `400` — missing required fields or `biddingDeadlineSeconds` below minimum
- `409` — duplicate task title within the last 60 seconds (idempotency guard)

---

## GET /tasks/{taskId}

Get the current state of a task.

**Owning component:** `TaskAuctioneerEndpoint` → `TaskAnnouncementEntity` (direct read)

**Path parameters**
- `taskId` — UUID returned by POST /tasks

**Response `200 OK`**
```json
{
  "taskId": "string",
  "title": "string",
  "description": "string",
  "requiredCapabilities": ["string"],
  "maxBudget": "decimal",
  "status": "ANNOUNCED | BIDDING | AWARDED | IN_PROGRESS | COMPLETED | FAILED | RENEGOTIATING",
  "bidCount": "integer",
  "winningBidId": "string | null",
  "winningContractorId": "string | null",
  "announcedAt": "ISO-8601",
  "awardedAt": "ISO-8601 | null",
  "completedAt": "ISO-8601 | null",
  "renegotiationCount": "integer"
}
```

**Errors**
- `404` — task not found

---

## POST /tasks/{taskId}/bids

Submit a contractor bid for an open task.

**Owning component:** `TaskAuctioneerEndpoint` → `TaskAnnouncementEntity.SubmitBid`

**Path parameters**
- `taskId` — UUID of the announced task

**Request body**
```json
{
  "contractorId": "string — required, must be a registered contractor",
  "costEstimate": "decimal — required, positive value",
  "capacityNote": "string | null — optional free text"
}
```

**Response `201 Created`**
```json
{
  "bidId": "string — UUID",
  "taskId": "string",
  "contractorId": "string",
  "submittedAt": "ISO-8601"
}
```

**Errors**
- `400` — missing fields or non-positive costEstimate
- `404` — task not found
- `409` — bidding window closed (task not in BIDDING status)
- `422` — contractor not registered or is throttled

---

## GET /tasks/{taskId}/bids

List all bids submitted for a task.

**Owning component:** `TaskAuctioneerEndpoint` → `TaskAnnouncementEntity` (read bids from entity state)

**Response `200 OK`**
```json
{
  "taskId": "string",
  "bids": [
    {
      "bidId": "string",
      "contractorId": "string",
      "costEstimate": "decimal",
      "capacityNote": "string | null",
      "submittedAt": "ISO-8601",
      "guardrailPassed": "boolean | null — null until guardrail runs",
      "evaluatorRationale": "string | null — null until evaluator runs"
    }
  ]
}
```

**Errors**
- `404` — task not found

---

## GET /auctions/active

Server-sent event stream of active task announcements. Emits the current snapshot on connect, then a delta event on each state change.

**Owning component:** `TaskAuctioneerEndpoint` → `ActiveAuctionsView` (SSE subscription)

**SSE event format**
```
event: auction-snapshot
data: {"auctions": [...]}

event: auction-updated
data: {"taskId": "string", "title": "string", "status": "string", "bidCount": integer, "announcedAt": "ISO-8601"}

event: auction-closed
data: {"taskId": "string", "status": "AWARDED | RENEGOTIATING | COMPLETED | FAILED"}
```

Initial `auction-snapshot` event contains all currently open auctions (status ANNOUNCED or BIDDING). Subsequent events are per-task deltas. The stream does not close; clients should reconnect on disconnect.

---

## POST /contractors/register

Register a contractor agent in the pool.

**Owning component:** `TaskAuctioneerEndpoint` → `ContractorRegistryEntity.RegisterContractor`

**Request body**
```json
{
  "name": "string — required",
  "capabilities": ["string — required, at least one"],
  "contractorId": "string | null — if omitted, service assigns a UUID"
}
```

**Response `201 Created`**
```json
{
  "contractorId": "string",
  "name": "string",
  "capabilities": ["string"],
  "performanceScore": 50,
  "throttled": false,
  "registeredAt": "ISO-8601"
}
```

New contractors start at `performanceScore: 50` (neutral). Score adjusts after first task completion.

**Errors**
- `400` — missing name or empty capabilities list
- `409` — contractorId already registered

---

## GET /contractors/{contractorId}

Get a contractor's current profile and performance record.

**Owning component:** `TaskAuctioneerEndpoint` → `ContractorRegistryEntity` (direct read)

**Response `200 OK`**
```json
{
  "contractorId": "string",
  "name": "string",
  "capabilities": ["string"],
  "performanceScore": "integer 0–100",
  "throttled": "boolean",
  "completedTasks": "integer",
  "registeredAt": "ISO-8601",
  "recentScores": [
    {
      "taskId": "string",
      "score": "integer",
      "feedback": "string",
      "scoredAt": "ISO-8601"
    }
  ]
}
```

`recentScores` returns the last five performance records, newest first.

**Errors**
- `404` — contractor not found

---

## GET /contractors/leaderboard

Server-sent event stream of contractors ranked by performance score, descending.

**Owning component:** `TaskAuctioneerEndpoint` → `ContractorLeaderboardView` (SSE subscription)

**SSE event format**
```
event: leaderboard-snapshot
data: {"contractors": [...]}

event: contractor-updated
data: {"contractorId": "string", "name": "string", "performanceScore": integer, "throttled": boolean, "completedTasks": integer, "rank": integer}

event: contractor-throttled
data: {"contractorId": "string", "performanceScore": integer, "throttledAt": "ISO-8601"}
```

Initial `leaderboard-snapshot` contains all registered contractors sorted by score. Subsequent events are per-contractor deltas. The stream does not close; clients should reconnect on disconnect.
