# Data model — ad-spend-payment-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CampaignBrief` | `title` | `String` | no | Campaign title submitted by the user. |
| | `brandVoice` | `String` | no | Brand voice descriptor (e.g., "direct, technical"). |
| | `targetAudience` | `String` | no | Audience description (e.g., "JVM developers"). |
| | `adFormat` | `String` | no | Ad format identifier (e.g., "banner-728x90"). |
| | `budgetWei` | `long` | no | Total campaign budget expressed in wei. |
| | `requestedBy` | `String` | no | Email or identifier of the submitter. |
| `CreativeLedger` | `facts` | `List<String>` | no | Facts the planner believes are known about the campaign. |
| | `missing` | `List<String>` | no | Facts the planner believes are still needed. |
| | `plan` | `List<String>` | no | Ordered list of plan steps (3–6). |
| | `currentDispatch` | `Optional<DispatchDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `DispatchDecision` | `executor` | `ExecutorKind` | no | Which executor runs the task (`COPYWRITER` or `PAYMENT`). |
| | `task` | `String` | no | One-sentence task description. |
| | `rationale` | `String` | no | One-sentence justification. |
| | `proposedSpendWei` | `Optional<Long>` | yes | Populated only when `executor = PAYMENT`; checked by the guardrail. |
| `CreativeResult` | `executor` | `ExecutorKind` | no | Executor that ran the task. |
| | `task` | `String` | no | Echo of the task. |
| | `ok` | `boolean` | no | True if the executor could fulfil the task. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `CreativeEntry` | `attempt` | `int` | no | 1-based attempt count for this `(executor, task)` pair. |
| | `executor` | `ExecutorKind` | no | Executor that ran the task. |
| | `task` | `String` | no | The task text. |
| | `verdict` | `CreativeVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED. |
| | `sanitizedContent` | `String` | no | Scrubbed result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `PaymentInstruction` | `placementId` | `String` | no | Unique identifier for this placement. |
| | `destinationWallet` | `String` | no | Target wallet address (must be on the allow-list). |
| | `amountWei` | `long` | no | Spend in wei (must be ≤ remaining budget). |
| | `token` | `String` | no | `ETH` or `USDC`. |
| | `memo` | `String` | no | Human-readable placement description passed to the tool. |
| `PaymentResult` | `placementId` | `String` | no | Echo of `PaymentInstruction.placementId`. |
| | `ok` | `boolean` | no | True if the on-chain call succeeded. |
| | `txHash` | `Optional<String>` | yes | Simulated transaction hash (`0x` prefixed hex). |
| | `failureReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| | `executedAt` | `Instant` | no | When the payment was executed. |
| `PlacementEntry` | `placementId` | `String` | no | Placement identifier. |
| | `task` | `String` | no | Task description from the dispatch decision. |
| | `approvalStatus` | `ApprovalStatus` | no | PENDING / APPROVED / REJECTED. |
| | `approvedBy` | `Optional<String>` | yes | Populated after `ApprovalGranted`. |
| | `approvedAt` | `Optional<Instant>` | yes | Populated after `ApprovalGranted`. |
| | `paymentResult` | `Optional<PaymentResult>` | yes | Populated after the executor returns. |
| | `recordedAt` | `Instant` | no | When the entry was first written. |
| `PaymentLedger` | `entries` | `List<PlacementEntry>` | no | Append-only. |
| `CampaignReport` | `summary` | `String` | no | 60–120 word summary. |
| | `placements` | `List<PlacementEntry>` | no | All placement entries at completion time. |
| | `totalSpentWei` | `long` | no | Sum of approved and successful payment amounts. |
| | `producedAt` | `Instant` | no | When the planner produced the report. |
| `Campaign` (entity state) | `campaignId` | `String` | no | Unique id. |
| | `title` | `String` | no | Campaign title. |
| | `requestedBy` | `String` | no | Submitter identifier. |
| | `budgetWei` | `long` | no | Total campaign budget in wei. |
| | `status` | `CampaignStatus` | no | See enum. |
| | `creativeLedger` | `Optional<CreativeLedger>` | yes | Populated after `CampaignPlanned`. |
| | `paymentLedger` | `Optional<PaymentLedger>` | yes | Populated after first `PaymentApprovalRequested`. |
| | `report` | `Optional<CampaignReport>` | yes | Populated after `CampaignCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `CampaignFailed` / `CampaignFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `CampaignHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `CampaignCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the campaign reached a terminal state. |
| `ApprovalRequest` (entity state) | `campaignId` | `String` | no | Parent campaign. |
| | `placementId` | `String` | no | Placement being approved. |
| | `task` | `String` | no | Task description. |
| | `amountWei` | `long` | no | Proposed spend. |
| | `destinationWallet` | `String` | no | Target wallet. |
| | `token` | `String` | no | `ETH` or `USDC`. |
| | `status` | `ApprovalStatus` | no | PENDING / APPROVED / REJECTED. |
| | `approvedBy` | `Optional<String>` | yes | Set on `ApprovalGranted`. |
| | `decidedAt` | `Optional<Instant>` | yes | Set on `ApprovalGranted` or `ApprovalRejected`. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextCampaignStep` | (sealed interface) | — | — | Permits `Continue(DispatchDecision)`, `Replan(CreativeLedger revised)`, `Complete(CampaignReport stub)`, `Fail(String reason)`. |

## Enums

- `ExecutorKind` → `COPYWRITER`, `PAYMENT`.
- `CreativeVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`.
- `ApprovalStatus` → `PENDING`, `APPROVED`, `REJECTED`.
- `CampaignStatus` → `PLANNING`, `EXECUTING`, `AWAITING_APPROVAL`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`CampaignEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CampaignCreated` | `campaignId, title, budgetWei, requestedBy, createdAt` | → PLANNING |
| `CampaignPlanned` | `ledger: CreativeLedger` | → EXECUTING |
| `CreativeDispatched` | `dispatch: DispatchDecision` | no status change; sets `creativeLedger.currentDispatch`. |
| `CreativeBlocked` | `attempt, dispatch, blocker` | no status change; appends a `CreativeEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `CreativeRecorded` | `entry: CreativeEntry` | no status change; appends to creative ledger history. |
| `PaymentApprovalRequested` | `placementId, amountWei, destinationWallet, token` | → AWAITING_APPROVAL; appends a `PlacementEntry` with `approvalStatus = PENDING`. |
| `PaymentApproved` | `placementId, approvedBy, approvedAt` | → EXECUTING; updates matching `PlacementEntry.approvalStatus = APPROVED`. |
| `PaymentRejected` | `placementId, rejectedBy, reason` | → EXECUTING; updates matching `PlacementEntry.approvalStatus = REJECTED`. |
| `PaymentRecorded` | `placementId, paymentResult: PaymentResult` | no status change; updates matching `PlacementEntry.paymentResult`. |
| `LedgerRevised` | `ledger: CreativeLedger` | no status change; replaces `creativeLedger`. |
| `CampaignCompleted` | `report: CampaignReport` | → COMPLETED, `finishedAt = now` |
| `CampaignFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `CampaignHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `CampaignFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`ApprovalEntity`)

| Event | Payload |
|---|---|
| `ApprovalRequested` | `campaignId, placementId, amountWei, destinationWallet, token, requestedAt` |
| `ApprovalGranted` | `approvedBy, grantedAt` |
| `ApprovalRejected` | `rejectedBy, reason, rejectedAt` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`CampaignQueue`)

| Event | Payload |
|---|---|
| `BriefSubmitted` | `campaignId, title, budgetWei, requestedBy, submittedAt` |

## View row

`CampaignRow` mirrors `Campaign` minus the full creative plan and heavy ledger payloads — `creativeLedger.plan` is omitted, `paymentLedger.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each `sanitizedContent` is capped at 240 characters. Every nullable field on the row record is declared `Optional<T>` (Lesson 6). The UI fetches the full campaign by id on row expand via `GET /api/campaigns/{id}`.
