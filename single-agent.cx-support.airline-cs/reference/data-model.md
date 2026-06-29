# Data model — airline-cs

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BookingChange` | `changeType` | `String` | no | One of `SEAT`, `DATE`, `FLIGHT`, `CANCEL`. |
| | `newValue` | `String` | no | The target value (e.g., `"14A"`, `"2026-07-15"`, `"AA456"`). |
| | `reason` | `String` | no | Customer-stated reason for the change. |
| `ServiceRequest` | `requestId` | `String` | no | UUID minted by `ServiceEndpoint`. |
| | `bookingRef` | `String` | no | Airline booking reference; may be empty for complaint-only flows. |
| | `rawMessage` | `String` | no | Pre-sanitization customer message. Audit-only. |
| | `submittedBy` | `String` | no | Session identifier. |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `redactedMessage` | `String` | no | PII redacted; this is what the agent reasons over. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","ffp","passport-token"]`. |
| `ToolCall` | `toolName` | `String` | no | Name of the tool invoked (e.g., `"searchBooking"`). |
| | `arguments` | `String` | no | JSON-serialized arguments. |
| | `result` | `String` | no | JSON-serialized tool result or error. |
| | `calledAt` | `Instant` | no | When the agent issued the call. |
| `ConfirmationRequest` | `proposedChange` | `String` | no | Human-readable description shown to the customer. |
| | `confirmationToken` | `String` | no | Opaque token issued by the workflow; validated by `ModificationGuardrail`. |
| `ServiceOutcome` | `status` | `OutcomeStatus` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `toolCallLog` | `List<ToolCall>` | no | All tool calls in call order. |
| | `caseNumber` | `String` | yes | Non-null only when `fileComplaint` was called. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `ServiceRequestState` | `requestId` | `String` | no | — |
| | `request` | `Optional<ServiceRequest>` | yes | Populated after `RequestReceived`. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `pendingConfirmation` | `Optional<ConfirmationRequest>` | yes | Populated after `ConfirmationRequested`; cleared after `ConfirmationReceived` or `RequestCancelled`. |
| | `outcome` | `Optional<ServiceOutcome>` | yes | Populated after `RequestCompleted`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ServiceRequestState` is `Optional<T>`. The view table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`OutcomeStatus`: `RESOLVED`, `CANCELLED`, `FAILED`, `NEEDS_FOLLOWUP`.

`RequestStatus`: `RECEIVED`, `SANITIZED`, `PROCESSING`, `AWAITING_CONFIRMATION`, `COMPLETED`, `CANCELLED`, `FAILED`.

## Events (`RequestEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RequestReceived` | `request` | → RECEIVED |
| `MessageSanitized` | `sanitized` | → SANITIZED |
| `ProcessingStarted` | — | → PROCESSING |
| `ConfirmationRequested` | `confirmation` | → AWAITING_CONFIRMATION |
| `ConfirmationReceived` | `token: String` | → PROCESSING |
| `RequestCompleted` | `outcome` | → COMPLETED (terminal happy) |
| `RequestCancelled` | — | → CANCELLED (terminal) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ServiceRequestState.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RequestRow` mirrors `ServiceRequestState` minus `request.rawMessage` (the audit log keeps that). The UI fetches the raw message on demand via `GET /api/requests/{id}` and reads `request.rawMessage` from the JSON.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM request_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ServiceTasks.java`)

```java
public final class ServiceTasks {
  public static final Task<ServiceOutcome> HANDLE_SERVICE_REQUEST = Task
      .name("Handle service request")
      .description("Reason over the customer's request using available booking tools and return a ServiceOutcome")
      .resultConformsTo(ServiceOutcome.class);

  private ServiceTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## BookingService responses (non-entity types)

These are plain Java records returned by `BookingService`. They are not persisted to the entity; they appear only in `ToolCall.result` strings.

| Record | Key fields |
|---|---|
| `BookingRecord` | `bookingRef`, `passengerName` (redacted in logs), `seat`, `fareClass`, `flightNumber`, `departureDate`, `upgradeEligible: boolean`, `cancellationEligible: boolean` |
| `ModificationResult` | `success: boolean`, `newSeat` / `newDate` / `newFlight`, `confirmationNumber` |
| `ComplaintReceipt` | `caseNumber`, `category`, `submittedAt` |
