# Data model — akka-async-agent-mesh

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DispatchRequest` | `requestId` | `String` | no | Id assigned at submission. |
| | `fromAgent` | `String` | no | Sending agent id. |
| | `toAgent` | `String` | no | Recipient agent id. |
| | `topic` | `String` | no | Short category label for the message. |
| | `body` | `String` | no | Full message text. |
| | `requestedAt` | `Instant` | no | When the request was submitted. |
| `MessagePayload` | `messageId` | `String` | no | Assigned by the dispatch tool. |
| | `fromAgent` | `String` | no | Sending agent id. |
| | `toAgent` | `String` | no | Recipient agent id. |
| | `topic` | `String` | no | Short category label. |
| | `body` | `String` | no | Full message text (may be sanitized by S1). |
| | `composedAt` | `Instant` | no | When the dispatcher agent composed the payload. |
| `DispatchResult` | `messageId` | `String` | no | The created message id (empty string if blocked). |
| | `deliveryReceiptId` | `String` | no | The receipt id from the tool (empty string if blocked). |
| | `blockedReason` | `Optional<String>` | yes | Guardrail refusal reason; empty on success. |
| `MemoryEntry` | `entryId` | `String` | no | Unique id. |
| | `agentId` | `String` | no | Owning agent id. |
| | `tag` | `String` | no | Short category label for the fact. |
| | `content` | `String` | no | The fact text. |
| | `recordedAt` | `Instant` | no | When the entry was added. |
| `ProcessingResult` | `messageId` | `String` | no | The message being processed. |
| | `processorAgent` | `String` | no | The agent id that processed the message. |
| | `summary` | `String` | no | One-sentence description of what was processed. |
| | `newMemory` | `List<MemoryEntry>` | no | New facts to persist (may be empty). |
| | `followUp` | `Optional<DispatchRequest>` | yes | Follow-up message to dispatch; empty when none. |
| `DeliveryReceipt` | `receiptId` | `String` | no | Unique id. |
| | `messageId` | `String` | no | The message this receipt acknowledges. |
| | `fromAgent` | `String` | no | Original sender. |
| | `toAgent` | `String` | no | Recipient agent. |
| | `acknowledgedAt` | `Instant` | no | When the tool returned the acknowledgement. |

## Entity state — `Message` (`MessageEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `messageId` | `String` | no | Assigned by the dispatch tool. |
| `requestId` | `String` | no | The originating `DispatchRequest` id. |
| `fromAgent` | `String` | no | Sending agent id. |
| `toAgent` | `String` | no | Recipient agent id. |
| `topic` | `String` | no | Short category label. |
| `body` | `String` | no | Sanitized message body (PII replaced). |
| `status` | `MessageStatus` | no | See enum. |
| `deliveryReceiptId` | `Optional<String>` | yes | Populated on `DELIVERED`. |
| `deliveredAt` | `Optional<Instant>` | yes | When the receipt was recorded. |
| `processingSummary` | `Optional<String>` | yes | Populated on `PROCESSED`. |
| `blockedReason` | `Optional<String>` | yes | Guardrail refusal reason; populated on `SEND_BLOCKED`. |
| `processedAt` | `Optional<Instant>` | yes | When processing completed. |
| `staledAt` | `Optional<Instant>` | yes | When `StaleMessageMonitor` advanced the message to `STALE`. |
| `createdAt` | `Instant` | no | When `MessageDispatched` emitted. |

## Entity state — `AgentMemory` (`AgentMemoryEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `agentId` | `String` | no | Agent id. |
| `entries` | `List<MemoryEntry>` | no | All accumulated facts (may be empty). |
| `lastUpdatedAt` | `Optional<Instant>` | yes | When the last `MemoryEntryAdded` emitted. |

## Enums

`MessageStatus`: `DISPATCHED`, `DELIVERED`, `PROCESSING`, `PROCESSED`, `SEND_BLOCKED`, `STALE`.

## Events — `MessageEntity`

| Event | Payload | Transition |
|---|---|---|
| `MessageDispatched` | `messageId, requestId, fromAgent, toAgent, topic, body, createdAt` | → DISPATCHED |
| `MessageDelivered` | `deliveryReceiptId, deliveredAt` | DISPATCHED → DELIVERED |
| `MessageProcessingStarted` | `startedAt` | DELIVERED → PROCESSING |
| `MessageProcessed` | `summary, processedAt` | PROCESSING → PROCESSED |
| `MessageSendBlocked` | `reason, blockedAt` | (pre-create) → SEND_BLOCKED |
| `MessageStaled` | `staledAt` | DISPATCHED → STALE |

Note: `SEND_BLOCKED` messages are created by a separate path in the dispatch workflow; no `MessageEntity` transitions from `DISPATCHED` to `SEND_BLOCKED` — the entity is created directly in the blocked state when the guardrail fires.

## Events — `AgentMemoryEntity`

| Event | Payload |
|---|---|
| `MemoryEntryAdded` | `MemoryEntry` |
| `MemoryCleared` | `clearedAt` |

## Events — `DeliveryReceiptEntity`

| Event | Payload |
|---|---|
| `ReceiptRecorded` | `DeliveryReceipt` |

## Events — `MessageBusEntity`

| Event | Payload |
|---|---|
| `DispatchRequestSubmitted` | `requestId, fromAgent, toAgent, topic, body, requestedAt` |
| `DispatchRequestBlocked` | `requestId, reason, blockedAt` |

## Key-value state — `SystemControl`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `halted` | `boolean` | no | Whether the mesh is frozen. |
| `haltedReason` | `Optional<String>` | yes | Operator-supplied reason. |
| `haltedBy` | `Optional<String>` | yes | Operator id. |
| `haltedAt` | `Optional<Instant>` | yes | When halted. |

## View row

`MessageRow` mirrors `Message` but keeps only `topic`, `processingSummary`, and `blockedReason` from the textual fields (the full `body` is not projected into the view for SSE efficiency). Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6). The UI fetches the full body from `GET /api/messages/{id}` when a user expands a card; the board view itself stays lightweight.
