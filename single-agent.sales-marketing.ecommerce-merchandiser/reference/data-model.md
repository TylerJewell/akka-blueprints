# Data model — merchandiser

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ProductScope` | `kind` | `ScopeKind` | no | `ALL`, `CATEGORY`, or `SKU_LIST`. |
| | `categorySlug` | `@Nullable String` | yes | Present when `kind == CATEGORY`. |
| | `skuList` | `@Nullable List<String>` | yes | Present when `kind == SKU_LIST`. |
| `MerchandisingObjective` | `proposalId` | `String` | no | UUID minted by `ProposalEndpoint`. |
| | `objectiveText` | `String` | no | Merchant's plain-language goal. |
| | `scope` | `ProductScope` | no | Which products the objective applies to. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ProductSnapshot` | `sku` | `String` | no | Stable product identifier. |
| | `name` | `String` | no | Display name. |
| | `currentDescription` | `String` | no | Current product description text. |
| | `currentPromotion` | `Optional<String>` | yes | Active promotion text, if any. |
| | `categorySlug` | `String` | no | Category this product belongs to. |
| | `price` | `double` | no | Current list price. |
| `CatalogContext` | `products` | `List<ProductSnapshot>` | no | Products matching the submitted scope. |
| | `categoryNames` | `List<String>` | no | All category names in the catalog (unfiltered). |
| | `activePromotionCount` | `int` | no | Count of currently active promotions across the catalog. |
| | `fetchedAt` | `Instant` | no | When `CatalogReader` built this snapshot. |
| `ChangeRecommendation` | `type` | `ChangeType` | no | Enum value. |
| | `targetRef` | `String` | no | SKU or `categorySlug`. |
| | `currentValue` | `String` | no | Current state of the field being changed. |
| | `proposedValue` | `String` | no | What the agent recommends instead. |
| | `rationale` | `String` | no | 1–2 sentences of evidence from the catalog. |
| | `confidence` | `Confidence` | no | `LOW`, `MEDIUM`, or `HIGH`. |
| `MerchandisingProposal` | `summary` | `String` | no | 2–4-sentence overview. |
| | `changes` | `List<ChangeRecommendation>` | no | One entry per proposed change. |
| | `overallConfidence` | `Confidence` | no | Minimum confidence across all changes. |
| | `proposedAt` | `Instant` | no | When the agent returned. |
| `ApprovalDecision` | `status` | `ApprovalStatus` | no | `APPROVED` or `REJECTED`. |
| | `decidedBy` | `String` | no | Merchant identifier. |
| | `note` | `Optional<String>` | yes | Optional rationale. |
| | `decidedAt` | `Instant` | no | When the merchant acted. |
| `Proposal` (entity state) | `proposalId` | `String` | no | — |
| | `objective` | `Optional<MerchandisingObjective>` | yes | Populated after `ObjectiveSubmitted`. |
| | `context` | `Optional<CatalogContext>` | yes | Populated after `CatalogContextLoaded`. |
| | `proposal` | `Optional<MerchandisingProposal>` | yes | Populated after `ProposalGenerated`. |
| | `approval` | `Optional<ApprovalDecision>` | yes | Populated after `ProposalApproved` or `ProposalRejected`. |
| | `status` | `ProposalStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ObjectiveSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Proposal` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ScopeKind`: `ALL`, `CATEGORY`, `SKU_LIST`.
`ChangeType`: `DESCRIPTION_UPDATE`, `PROMOTION_CREATE`, `PROMOTION_REMOVE`, `CATEGORY_REORDER`, `PRICE_NUDGE`.
`Confidence`: `LOW`, `MEDIUM`, `HIGH`.
`ApprovalStatus`: `APPROVED`, `REJECTED`.
`ProposalStatus`: `SUBMITTED`, `CONTEXT_LOADED`, `GENERATING`, `PENDING_APPROVAL`, `PUBLISHING`, `PUBLISHED`, `REJECTED`, `EXPIRED`, `FAILED`.

## Events (`ProposalEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ObjectiveSubmitted` | `objective` | → SUBMITTED |
| `CatalogContextLoaded` | `context` | → CONTEXT_LOADED |
| `GenerationStarted` | — | → GENERATING |
| `ProposalGenerated` | `proposal` | → PENDING_APPROVAL |
| `ProposalApproved` | `decision` | → PUBLISHING |
| `ProposalRejected` | `decision` | → REJECTED (terminal) |
| `ProposalPublished` | — | → PUBLISHED (terminal happy) |
| `ProposalExpired` | — | → EXPIRED (terminal) |
| `ProposalFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Proposal.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ProposalRow` mirrors `Proposal`. The view declares ONE query: `getAllProposals: SELECT * AS proposals FROM proposal_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ProposalTasks.java`)

```java
public final class ProposalTasks {
  public static final Task<MerchandisingProposal> GENERATE_PROPOSAL = Task
      .name("Generate merchandising proposal")
      .description("Read the attached catalog snapshot and produce a MerchandisingProposal with ChangeRecommendations for the merchant's objective")
      .resultConformsTo(MerchandisingProposal.class);

  private ProposalTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
