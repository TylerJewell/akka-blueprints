# Data model — sales-offer-generator

## `Offer` (OfferEntity state + OffersView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Offer UUID (workflow id) |
| `brief` | `Optional<String>` | yes | Sanitized customer brief |
| `status` | `OfferStatus` | no | Lifecycle state |
| `analyzedAt` | `Optional<Instant>` | yes | When needs were analyzed |
| `needs` | `Optional<CustomerNeeds>` | yes | Structured needs |
| `matchedAt` | `Optional<Instant>` | yes | When products were matched |
| `products` | `Optional<ProductMatch>` | yes | Matched products |
| `pricedAt` | `Optional<Instant>` | yes | When pricing was built |
| `pricing` | `Optional<PricingStrategy>` | yes | Pricing strategy |
| `composedAt` | `Optional<Instant>` | yes | When the offer was composed |
| `offerTitle` | `Optional<String>` | yes | Offer document title |
| `offerBody` | `Optional<String>` | yes | Offer document body |
| `quotedTotal` | `Optional<Double>` | yes | Quoted total in the offer |
| `approvedAt` | `Optional<Instant>` | yes | When the offer passed policy |
| `rejectedAt` | `Optional<Instant>` | yes | When the offer failed policy |
| `rejectReason` | `Optional<String>` | yes | Policy failure reason |

Every nullable lifecycle field is `Optional<T>` (Lesson 6). `emptyState()` returns `Offer.initial("")` with no `commandContext()` reference (Lesson 3).

## `OfferStatus` enum

`QUEUED · ANALYZED · MATCHED · PRICED · COMPOSED · APPROVED · REJECTED`

## Agent result records

- `CustomerNeeds(String segment, List<String> painPoints, String budgetBand, List<String> desiredOutcomes)`
- `MatchedProduct(String sku, String name, double listPrice, String fitReason)`
- `ProductMatch(List<MatchedProduct> products)`
- `PricingStrategy(double subtotal, double discountPct, double total, String rationale)`
- `OfferDocument(String title, String body, double quotedTotal)`

## Events (OfferEntity)

| Event | Trigger |
|---|---|
| `NeedsAnalyzed(CustomerNeeds needs, Instant at)` | `analyzeStep` completes |
| `ProductsMatched(ProductMatch products, Instant at)` | `matchStep` completes |
| `OfferPriced(PricingStrategy pricing, Instant at)` | `priceStep` completes |
| `OfferComposed(String title, String body, double quotedTotal, Instant at)` | `composeStep` completes |
| `OfferApproved(Instant at)` | `reviewStep` passes `PricingPolicy` |
| `OfferRejected(String reason, Instant at)` | `reviewStep` fails `PricingPolicy` |

## InboundRequestQueue

- State: list of queued briefs (single instance `"default"`).
- Event: `BriefQueued(String brief)` — emitted by `enqueueBrief`.
- Consumed by `RequestConsumer`, which sanitizes the brief and starts an `OfferWorkflow`.

## OffersView row

The view row type is `Offer` (above). One query: `getAllOffers` — `SELECT * AS offers FROM offers_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).
