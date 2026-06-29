# Data model — personalized-shopper

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PreferenceProfile` | `sessionId` | `String` | no | UUID minted by `ShoppingEndpoint`. |
| | `preferredCategories` | `List<String>` | no | Product categories the shopper wants. |
| | `preferredBrands` | `List<String>` | no | Brand names (may be empty). |
| | `minPrice` | `double` | no | Inclusive lower price bound. |
| | `maxPrice` | `double` | no | Inclusive upper price bound. |
| | `excludedCategories` | `List<String>` | no | Hard-excluded categories. |
| | `freeTextNote` | `String` | no | Shopper's open-ended note (may be empty string). |
| | `shopperEmail` | `String` | no | PII — stripped by `ProfileSanitizer` before LLM. |
| | `shopperPhone` | `String` | no | PII — stripped by `ProfileSanitizer` before LLM. |
| | `loyaltyCardNumber` | `String` | no | PII — stripped by `ProfileSanitizer` before LLM. |
| | `shopperName` | `String` | no | PII — stripped by `ProfileSanitizer` before LLM. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedProfile` | `preferredCategories` | `List<String>` | no | Copied from PreferenceProfile. |
| | `preferredBrands` | `List<String>` | no | Copied from PreferenceProfile. |
| | `minPrice` | `double` | no | Copied from PreferenceProfile. |
| | `maxPrice` | `double` | no | Copied from PreferenceProfile. |
| | `excludedCategories` | `List<String>` | no | Copied from PreferenceProfile. |
| | `freeTextNote` | `String` | no | PII stripped from free text. |
| | `piiCategoriesStripped` | `List<String>` | no | e.g. `["email","phone","loyalty-card","person-name"]`. |
| `CatalogProduct` | `productId` | `String` | no | Stable product identifier. |
| | `productName` | `String` | no | Display name. |
| | `category` | `String` | no | e.g. `"electronics"`. |
| | `brand` | `String` | no | Brand name. |
| | `price` | `double` | no | Current price. |
| | `inStock` | `boolean` | no | Whether available for immediate purchase. |
| | `currentSeason` | `boolean` | no | Whether seasonally relevant at time of catalog snapshot. |
| | `tags` | `List<String>` | no | e.g. `["energy-efficient","smart-home"]`. |
| `ProductRecommendation` | `rank` | `int` | no | 1 = best match. |
| | `productId` | `String` | no | MUST match a productId in the provided catalog. |
| | `productName` | `String` | no | Copied from catalog entry. |
| | `category` | `String` | no | Copied from catalog entry. |
| | `price` | `double` | no | Copied from catalog entry. |
| | `rationale` | `String` | no | 1–2 sentence explanation. |
| | `confidence` | `ConfidenceTier` | no | Enum value. |
| `RecommendationSet` | `outcome` | `RecommendationOutcome` | no | Enum value. |
| | `explanation` | `String` | no | 1–3 sentences. |
| | `recommendations` | `List<ProductRecommendation>` | no | Ranked list, may be empty on NO_MATCH. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `FreshnessResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `scoredAt` | `Instant` | no | When `FreshnessScorer` finished. |
| `ShoppingSession` (entity state) | `sessionId` | `String` | no | — |
| | `profile` | `Optional<PreferenceProfile>` | yes | Populated after `SessionStarted`. |
| | `sanitized` | `Optional<SanitizedProfile>` | yes | Populated after `ProfileSanitized`. |
| | `recommendations` | `Optional<RecommendationSet>` | yes | Populated after `RecommendationsRecorded`. |
| | `freshness` | `Optional<FreshnessResult>` | yes | Populated after `FreshnessScored`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionStarted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ShoppingSession` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ConfidenceTier`: `HIGH`, `MEDIUM`, `LOW`.
`RecommendationOutcome`: `MATCHED`, `PARTIAL_MATCH`, `NO_MATCH`.
`SessionStatus`: `STARTED`, `PROFILE_SANITIZED`, `RECOMMENDING`, `RECOMMENDATIONS_READY`, `FRESHNESS_SCORED`, `FAILED`.

## Events (`ShoppingSessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionStarted` | `profile` | → STARTED |
| `ProfileSanitized` | `sanitized` | → PROFILE_SANITIZED |
| `RecommendationStarted` | — | → RECOMMENDING |
| `RecommendationsRecorded` | `recommendations` | → RECOMMENDATIONS_READY |
| `FreshnessScored` | `freshness` | → FRESHNESS_SCORED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ShoppingSession.initial("")` with all `Optional` fields as `Optional.empty()` and `status = STARTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `ShoppingSession` minus `profile.shopperEmail`, `profile.shopperPhone`, `profile.loyaltyCardNumber`, and `profile.shopperName` — the audit log retains those fields on the entity. The UI fetches the full entity on demand via `GET /api/sessions/{id}` to access raw PII for audit purposes.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM recommendation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ShoppingTasks.java`)

```java
public final class ShoppingTasks {
  public static final Task<RecommendationSet> RECOMMEND_PRODUCTS = Task
      .name("Recommend products")
      .description("Read the attached preference profile and the product catalog, then produce a ranked RecommendationSet")
      .resultConformsTo(RecommendationSet.class);

  private ShoppingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## FreshnessScorer rubric

`FreshnessScorer` is a pure deterministic function. Inputs: `RecommendationSet` and `List<CatalogProduct>`. Output: `FreshnessResult`.

Scoring logic:
- Start at 3.
- +1 for each recommended product where `inStock = true` (capped at +2 total).
- +1 if all top-3 ranked products are `currentSeason = true`.
- -1 if the rank-1 product is `inStock = false`.
- Clamp final score to `[1, 5]`.
- On `NO_MATCH` (empty recommendations list), score = 1 with rationale "No products were recommended."
