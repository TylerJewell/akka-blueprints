# Data Model: Agent-Initiated Payment Settlement

## Java records

### `PaymentTaskRequest`

| Field | Type | Nullable | Description |
|---|---|---|---|
| `taskId` | `String` | No | Caller-assigned unique task identifier |
| `description` | `String` | No | Natural-language task description for the agent |
| `budgetCents` | `long` | No | Total spend ceiling for this task in cents |
| `approvalThresholdCents` | `long` | No | Payments at or above this amount require human approval |

### `PaymentIntent`

| Field | Type | Nullable | Description |
|---|---|---|---|
| `paymentId` | `String` | No | Agent-assigned unique identifier for this payment |
| `targetApi` | `String` | No | External API endpoint or service identifier |
| `amountCents` | `long` | No | Requested payment amount in cents |
| `purpose` | `String` | No | Agent's one-sentence rationale for this payment |
| `requestedAt` | `Instant` | No | Timestamp when the agent submitted the intent |

### `ApprovalDecision`

| Field | Type | Nullable | Description |
|---|---|---|---|
| `paymentId` | `String` | No | The payment being decided |
| `decision` | `ApprovalStatus` | No | `APPROVE` or `REJECT` |
| `reviewer` | `String` | No | Identity of the human reviewer |
| `decidedAt` | `Instant` | No | Timestamp of the decision |
| `note` | `Optional<String>` | Yes | Optional reviewer note |

### `PaymentRecord`

| Field | Type | Nullable | Description |
|---|---|---|---|
| `paymentId` | `String` | No | Payment identifier |
| `targetApi` | `String` | No | Target service |
| `amountCents` | `long` | No | Amount in cents |
| `status` | `PaymentStatus` | No | Current payment status |
| `purpose` | `String` | No | Agent's rationale |
| `approvalRequired` | `boolean` | No | Whether this payment required HITL |
| `requestedAt` | `Instant` | No | When the agent submitted the intent |
| `reviewedBy` | `Optional<String>` | Yes | Reviewer identity if applicable |
| `settledAt` | `Optional<Instant>` | Yes | Timestamp of settlement or rejection |

### `PaymentTask` (entity state)

| Field | Type | Nullable | Description |
|---|---|---|---|
| `taskId` | `String` | No | Task identifier |
| `description` | `String` | No | Task description |
| `budgetCents` | `long` | No | Total budget |
| `approvalThresholdCents` | `long` | No | Approval threshold |
| `accumulatedSpentCents` | `long` | No | Sum of all settled payment amounts |
| `status` | `TaskStatus` | No | Current task status |
| `payments` | `List<PaymentRecord>` | No | All payment records (empty list if none) |
| `createdAt` | `Instant` | No | Task creation timestamp |
| `completedAt` | `Optional<Instant>` | Yes | Completion or halt timestamp |

### `PaymentLedgerRow` (view row type)

| Field | Type | Nullable | Description |
|---|---|---|---|
| `taskId` | `String` | No | Task identifier |
| `budgetCents` | `long` | No | Total budget |
| `spentCents` | `long` | No | Accumulated settled spend |
| `remainingCents` | `long` | No | `budgetCents - spentCents` |
| `status` | `TaskStatus` | No | Current task status |
| `totalPayments` | `int` | No | Count of all payment records |
| `pendingApprovals` | `int` | No | Count of `PENDING_APPROVAL` payments |
| `createdAt` | `Instant` | No | Task creation timestamp |

---

## Events on `PaymentTaskEntity`

| Event class | Trigger | Key fields |
|---|---|---|
| `TaskCreated` | `submitTask` command accepted | `taskId`, `description`, `budgetCents`, `approvalThresholdCents`, `createdAt` |
| `AgentStarted` | Agent session begins iteration | `taskId`, `startedAt` |
| `PaymentIntentSubmitted` | Agent calls `submitPaymentIntent` tool | `paymentId`, `targetApi`, `amountCents`, `purpose`, `requestedAt` |
| `PaymentHeld` | Intent amount ≥ threshold | `paymentId`, `amountCents`, `heldAt` |
| `PaymentApproved` | Workflow receives `APPROVE` | `paymentId`, `reviewer`, `decidedAt`, `note` |
| `PaymentRejected` | Workflow receives `REJECT` or timeout | `paymentId`, `reviewer` (null on timeout), `decidedAt`, `reason` |
| `PaymentSettled` | Payment executed (below threshold or post-approval) | `paymentId`, `amountCents`, `settledAt` |
| `BudgetCapReached` | `accumulatedSpentCents + amountCents >= budgetCents` | `taskId`, `accumulatedSpentCents`, `budgetCents`, `haltedAt` |
| `TaskCompleted` | Agent signals completion | `taskId`, `accumulatedSpentCents`, `totalPayments`, `completedAt` |
| `TaskFailed` | Unrecoverable agent session error | `taskId`, `reason`, `failedAt` |

---

## Status enums

### `TaskStatus`

| Value | Meaning |
|---|---|
| `PENDING` | Task created; agent not yet started |
| `ACTIVE` | Agent is running; payments may be settling |
| `AWAITING_APPROVAL` | At least one payment is held for human review |
| `BUDGET_EXCEEDED` | Budget cap reached; no further payments accepted |
| `COMPLETED` | Agent finished all intended payments |
| `FAILED` | Unrecoverable error |

### `PaymentStatus`

| Value | Meaning |
|---|---|
| `PENDING_APPROVAL` | Payment held; awaiting human decision |
| `APPROVED` | Human approved; payment settling |
| `REJECTED` | Human rejected or approval timed out |
| `SETTLED` | Payment executed |
| `CANCELLED` | Payment cancelled due to task halt |

### `ApprovalStatus`

| Value | Meaning |
|---|---|
| `APPROVE` | Reviewer approves execution |
| `REJECT` | Reviewer rejects execution |

---

## Notes on Lesson 6

All fields that may be absent use `Optional<T>`. The following fields are `Optional`:
- `ApprovalDecision.note`
- `PaymentRecord.reviewedBy`
- `PaymentRecord.settledAt`
- `PaymentTask.completedAt`

No record field uses raw `null`. The entity state initializes `Optional` fields with `Optional.empty()`.
