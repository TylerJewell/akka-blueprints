# Data model — whatsapp-fintech-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundMessage` | `messageId` | `String` | no | UUID minted by `MessageEndpoint`. |
| | `customerId` | `String` | no | Customer identifier from the channel. |
| | `accountId` | `String` | no | Primary account for context. |
| | `rawText` | `String` | no | Pre-sanitization message body. Audit-only. |
| | `channelMessageId` | `String` | no | WhatsApp-side message reference. |
| | `receivedAt` | `Instant` | no | When the endpoint received the message. |
| `SanitizedMessage` | `redactedText` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["phone","account-number","person-name"]`. |
| `ToolCall` | `toolName` | `String` | no | e.g. `"get_balance"`, `"transfer_funds"`. |
| | `args` | `Map<String, String>` | no | Arguments the agent supplied. |
| | `result` | `String` | no | What the tool returned. |
| `AgentResponse` | `intent` | `Intent` | no | Enum value. |
| | `responseText` | `String` | no | 1–3 plain-language sentences for the customer. |
| | `toolCallsMade` | `List<ToolCall>` | no | Zero or more tool calls. |
| | `pendingTransaction` | `Optional<PendingTransaction>` | yes | Present only for `FUND_TRANSFER` intent. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `PendingTransaction` | `fromAccountId` | `String` | no | Source account. |
| | `toAccountId` | `String` | no | Destination account (guardrail-validated). |
| | `amount` | `BigDecimal` | no | Transfer amount. |
| | `currency` | `String` | no | ISO 4217 currency code. |
| | `description` | `String` | no | Customer-supplied memo. |
| `HitlDecision` | `operatorId` | `String` | no | Operator who acted. |
| | `outcome` | `HitlOutcome` | no | `APPROVED` or `REJECTED`. |
| | `note` | `String` | no | Operator's free-text note. |
| | `decidedAt` | `Instant` | no | When the operator acted. |
| `Message` (entity state) | `messageId` | `String` | no | — |
| | `inbound` | `Optional<InboundMessage>` | yes | Populated after `MessageReceived`. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `response` | `Optional<AgentResponse>` | yes | Populated after `ResponseReady`. |
| | `hitlDecision` | `Optional<HitlDecision>` | yes | Populated after `TransactionApproved` or `TransactionRejected`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Message` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Intent`: `BALANCE_QUERY`, `TRANSACTION_HISTORY`, `FUND_TRANSFER`, `ACCOUNT_INFO`, `UNKNOWN`.

`HitlOutcome`: `APPROVED`, `REJECTED`.

`MessageStatus`: `RECEIVED`, `SANITIZED`, `RESPONDING`, `RESPONSE_READY`, `AWAITING_APPROVAL`, `EXECUTED`, `HITL_REJECTED`, `FAILED`.

## Events (`MessageEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageReceived` | `inbound` | → RECEIVED |
| `MessageSanitized` | `sanitized` | → SANITIZED |
| `ResponseStarted` | — | → RESPONDING |
| `ResponseReady` | `response` | → RESPONSE_READY |
| `ApprovalRequested` | `pendingTransaction` | → AWAITING_APPROVAL |
| `TransactionApproved` | `decision` | → (workflow resume) |
| `TransactionRejected` | `decision` | → (workflow resume) |
| `TransactionExecuted` | — | → EXECUTED (terminal happy) |
| `MessageFailed` | `reason: String` | → FAILED (terminal) |

`HITL_REJECTED` is a terminal state reached when `TransactionRejected` is followed by `executeStep` writing no further events (the workflow just sets the entity status). `emptyState()` returns `Message.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`MessageRow` mirrors `Message` minus `inbound.rawText` (the audit log keeps that). The UI fetches the raw message on demand via `GET /api/messages/{id}` and reads `inbound.rawText` from the JSON.

The view declares ONE query: `getAllMessages: SELECT * AS messages FROM message_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`MessageTasks.java`)

```java
public final class MessageTasks {
  public static final Task<AgentResponse> HANDLE_CUSTOMER_MESSAGE = Task
      .name("Handle customer message")
      .description("Classify the customer intent, call the appropriate tools, and return an AgentResponse with a reply text")
      .resultConformsTo(AgentResponse.class);

  private MessageTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
