# Data model — product-catalog-ad-generation

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ProductAttributes` | `category` | `String` | no | Product category slug (e.g. `"consumer-electronics"`). |
| | `pricingTier` | `String` | no | One of `"budget"`, `"mid-range"`, `"premium"`. |
| | `keyFeatures` | `List<String>` | no | Possibly empty when catalog entry is absent. |
| | `targetAudience` | `String` | no | Short description of the intended buyer. |
| `EnrichedProduct` | `productId` | `String` | no | Short stable id (`"prod-" + slugify(productName)`). |
| | `productName` | `String` | no | As submitted by the user. |
| | `attributes` | `ProductAttributes` | no | Output of `inferAttributes`. |
| | `enrichedAt` | `Instant` | no | When the ENRICH task returned. |
| `AdPlacement` | `type` | `String` | no | One of `"search"`, `"display"`, `"social"`. |
| | `copy` | `String` | no | The ad copy text for this placement. |
| | `charCount` | `int` | no | MUST equal `copy.length()`. Limits: search ≤ 130, display ≤ 300, social ≤ 280. |
| `AdDraft` | `headline` | `String` | no | MUST contain at least one brand element. |
| | `callToAction` | `String` | no | MUST match `[a-z0-9-]+`. |
| | `placements` | `List<AdPlacement>` | no | At least 2 entries for a passing compliance score. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `PolicyViolation` | `field` | `String` | no | Name of the failing field (e.g. `"headline"`, `"placements[display]"`). |
| | `rule` | `String` | no | Rule name (e.g. `"prohibited-word"`, `"placement-char-limit"`). |
| | `found` | `String` | no | What the guardrail observed (the offending word or count). |
| `ProductAd` | `jobId` | `String` | no | Links back to the originating `AdJobEntity`. |
| | `headline` | `String` | no | Approved headline, same brand constraints as `AdDraft`. |
| | `callToAction` | `String` | no | Approved CTA slug. |
| | `placements` | `List<AdPlacement>` | no | Final per-placement copy. |
| | `approvedAt` | `Instant` | no | When the REVIEW task returned. |
| `ComplianceResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `scoredAt` | `Instant` | no | When `BrandComplianceScorer` finished. |
| `BrandRejection` | `violations` | `List<PolicyViolation>` | no | Every failing rule in this response. Non-empty on reject. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `AdJobRecord` (entity state) | `jobId` | `String` | no | — |
| | `productName` | `Optional<String>` | yes | Populated after `AdJobCreated`. |
| | `enrichedProduct` | `Optional<EnrichedProduct>` | yes | Populated after `ProductEnriched`. |
| | `adDraft` | `Optional<AdDraft>` | yes | Populated after `AdDrafted`. |
| | `productAd` | `Optional<ProductAd>` | yes | Populated after `AdApproved`. |
| | `complianceResult` | `Optional<ComplianceResult>` | yes | Populated after `ComplianceScored`. |
| | `status` | `AdJobStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AdJobCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<BrandRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `AdJobRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AdJobStatus`: `CREATED`, `ENRICHING`, `ENRICHED`, `DRAFTING`, `DRAFTED`, `REVIEWING`, `APPROVED`, `SCORED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `BrandGuardrail`): `ENRICH`, `DRAFT`, `REVIEW`.

## Events (`AdJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AdJobCreated` | `productName: String` | → CREATED |
| `EnrichStarted` | — | → ENRICHING |
| `ProductEnriched` | `enrichedProduct: EnrichedProduct` | → ENRICHED |
| `DraftStarted` | — | → DRAFTING |
| `AdDrafted` | `adDraft: AdDraft` | → DRAFTED |
| `ReviewStarted` | — | → REVIEWING |
| `AdApproved` | `productAd: ProductAd` | → APPROVED |
| `ComplianceScored` | `complianceResult: ComplianceResult` | → SCORED (terminal happy) |
| `GuardrailRejected` | `phase, field, rule, found, rejectedAt` | no status change (audit-only) |
| `AdJobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AdJobRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AdJobRow` mirrors `AdJobRecord` exactly. The UI fetches the full row via `GET /api/ad-jobs/{id}` and streams updates via `GET /api/ad-jobs/sse`.

The view declares ONE query: `getAllAdJobs: SELECT * AS adJobs FROM ad_job_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AdTasks.java`)

```java
public final class AdTasks {
  public static final Task<EnrichedProduct> ENRICH_PRODUCT = Task
      .name("Enrich product")
      .description("Look up category and infer attributes for the product to build a complete EnrichedProduct record")
      .resultConformsTo(EnrichedProduct.class);

  public static final Task<AdDraft> DRAFT_AD = Task
      .name("Draft ad")
      .description("Compose headline, call-to-action, and placement copy for each ad placement type")
      .resultConformsTo(AdDraft.class);

  public static final Task<ProductAd> REVIEW_AD = Task
      .name("Review ad")
      .description("Check the draft, resolve any remaining brand issues, and produce the final ProductAd")
      .resultConformsTo(ProductAd.class);

  private AdTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Brand-policy rules (`BrandGuardrail` + `BrandComplianceScorer`)

Four rules, each worth one point in the compliance score (base of 1):

1. **Headline element** — `ProductAd.headline` (or `AdDraft.headline`) must contain at least one string from `src/main/resources/brand-policy/elements.json`.
2. **Prohibited words** — no copy field (`headline`, `callToAction`, any `AdPlacement.copy`) may include a word from `src/main/resources/brand-policy/prohibited-words.txt` (case-insensitive word-boundary match).
3. **CTA slug** — `callToAction` must match the regex `[a-z0-9-]+` (URL-safe slug, no spaces or special chars).
4. **Placement parity** — `placements.size() >= 2` AND every placement's `charCount` is within the limit for its type (`placement-limits.json`).

Score range 1–5. The rationale sentence names the first failing rule (or "all checks passed" when score = 5).
