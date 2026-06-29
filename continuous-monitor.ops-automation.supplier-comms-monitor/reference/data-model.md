# Data model — supplier-comms-monitor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PoFeedEvent` | `poId` | `String` | no | Unique PO identifier from the feed. |
| | `supplierId` | `String` | no | Supplier code. |
| | `supplierName` | `String` | no | Human-readable supplier name. |
| | `itemDescription` | `String` | no | Description of ordered item. |
| | `orderedQty` | `int` | no | Quantity on the PO line. |
| | `promisedDate` | `LocalDate` | no | Supplier's committed delivery date. |
| | `currency` | `String` | no | ISO 4217 code. |
| | `unitPrice` | `BigDecimal` | no | Unit price on the PO. |
| | `feedTimestamp` | `Instant` | no | When the PO appeared in the feed. |
| `RiskAssessment` | `tier` | `RiskTier` | no | One of `ON_TRACK`, `AT_RISK`, `CRITICAL`. |
| | `confidence` | `String` | no | `"high" \| "medium" \| "low"`. |
| | `rationale` | `String` | no | One short sentence. |
| | `riskFactors` | `List<String>` | no | 1–3 short factor labels. |
| `OutreachDraft` | `subjectLine` | `String` | no | `"Delivery Confirmation: …"`. |
| | `body` | `String` | no | 2–4 paragraphs. |
| | `requiresBuyerApproval` | `boolean` | no | `true` for CRITICAL or material changes. |
| | `draftedAt` | `Instant` | no | When the agent finished. |
| `BuyerDecision` | `approved` | `boolean` | no | `true` = send outreach, `false` = escalate. |
| | `decidedBy` | `String` | no | Buyer identifier. |
| | `escalationNote` | `Optional<String>` | yes | Required when `approved=false`. |
| | `decidedAt` | `Instant` | no | When the buyer acted. |
| `AccuracyScore` | `poId` | `String` | no | The PO being evaluated. |
| | `predictedTier` | `RiskTier` | no | What the agent predicted at assessment time. |
| | `actualOutcome` | `String` | no | `"on-time" \| "late" \| "cancelled" \| "partial-delivery"`. |
| | `predictionCorrect` | `boolean` | no | Whether prediction matched outcome. |
| | `evalRationale` | `String` | no | One sentence. |
| `PurchaseOrder` (entity state) | `poId` | `String` | no | — |
| | `feedEvent` | `PoFeedEvent` | no | Captured once at creation. |
| | `riskAssessment` | `Optional<RiskAssessment>` | yes | Populated after RiskAssessed. |
| | `outreachDraft` | `Optional<OutreachDraft>` | yes | Populated after OutreachDrafted. |
| | `buyerDecision` | `Optional<BuyerDecision>` | yes | Populated after BuyerApproved/Escalated. |
| | `accuracyScore` | `Optional<AccuracyScore>` | yes | Populated after AccuracyEvaluated. |
| | `status` | `PoStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When PoOpened was emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

## Enums

`RiskTier`: `ON_TRACK`, `AT_RISK`, `CRITICAL`.

`PoStatus`: `OPENED`, `RISK_ASSESSED`, `OUTREACH_DRAFTED`, `AWAITING_BUYER_APPROVAL`, `OUTREACH_SENT`, `PROCUREMENT_REVIEW`, `ON_TRACK_CONFIRMED`, `CLOSED`.

## Events (`PurchaseOrderEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PoOpened` | `feedEvent` | → OPENED |
| `RiskAssessed` | `riskAssessment` | → RISK_ASSESSED |
| `OutreachDrafted` | `outreachDraft` | → OUTREACH_DRAFTED (then conditional AWAITING_BUYER_APPROVAL) |
| `BuyerApproved` | `buyerDecision` | (no status change; precedes OutreachSent) |
| `BuyerEscalated` | `buyerDecision` | (no status change; precedes ProcurementReview) |
| `OutreachSent` | — | → OUTREACH_SENT (terminal path) |
| `ProcurementReview` | — | → PROCUREMENT_REVIEW (terminal path) |
| `OnTrackConfirmed` | — | → ON_TRACK_CONFIRMED (terminal path) |
| `PoClosed` | `closedAt` | → CLOSED |
| `AccuracyEvaluated` | `accuracyScore` | (no status change; populates accuracyScore) |

## Events (`PoFeedQueue`)

| Event | Payload |
|---|---|
| `PoFeedReceived` | `feedEvent` (the raw feed event — used by the audit log) |

## View row

`PoOrderRow` mirrors `PurchaseOrder`. The UI fetches the full record on-demand via `GET /api/orders/{id}`.
