# Data model — ad-creator-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ProductAttribute` | `name` | `String` | no | Short attribute label (e.g., "battery life"). |
| | `value` | `String` | no | Specific claim from the product page. |
| `ProductProfile` | `productName` | `String` | no | Display name of the product. |
| | `productUrl` | `String` | no | The URL the scrape phase fetched. |
| | `attributes` | `List<ProductAttribute>` | no | Possibly empty; J5 demonstrates the empty path. |
| | `targetAudience` | `String` | no | Audience description extracted from the product page. |
| | `brandConstraints` | `List<String>` | no | Copy-writing constraints embedded in the product fixture. |
| | `scrapedAt` | `Instant` | no | When the SCRAPE task returned. |
| `AdVariant` | `format` | `String` | no | `INSTAGRAM` / `FACEBOOK` / `SEARCH`. |
| | `headline` | `String` | no | Format-specific headline. |
| | `body` | `String` | no | Format-specific body copy. MUST reference at least one `ProductAttribute.name` (E1 rule 1). |
| | `callToAction` | `String` | no | Format-specific CTA (e.g., "Shop now"). |
| `AdCopy` | `variants` | `List<AdVariant>` | no | Exactly three entries — one per format. |
| | `primaryHeadline` | `String` | no | Matches the INSTAGRAM variant headline. |
| | `primaryBody` | `String` | no | Matches the INSTAGRAM variant body. |
| | `callToAction` | `String` | no | Primary CTA. Non-blank (E1 rule 2). |
| | `draftedAt` | `Instant` | no | When the COPY task returned. |
| `VisualSpec` | `imagePrompt` | `String` | no | Detailed image-generation prompt. MUST reference `productName` (E1 rule 3). |
| | `aspectRatio` | `String` | no | `1:1` / `4:3` / `3:1` per primary format. |
| | `styleGuidance` | `String` | no | High-level style notes for the visual. |
| | `generatedAt` | `Instant` | no | When the VISUAL task returned. |
| `AdPackage` | `title` | `String` | no | 1-line title derived from productName. |
| | `copy` | `AdCopy` | no | The complete copy result. |
| | `visual` | `VisualSpec` | no | The complete visual result. |
| | `assembledAt` | `Instant` | no | When `evalStep` assembled the package. |
| `BrandViolation` | `rule` | `String` | no | `disallowed_superlative` / `competitor_name` / `prohibited_category`. |
| | `offendingText` | `String` | no | The exact text fragment that triggered the rule. |
| | `suggestion` | `String` | no | Rewrite guidance returned by `BrandSafetyGuardrail`. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `AdQualityScorer` finished. |
| `AdJobRecord` (entity state) | `adJobId` | `String` | no | — |
| | `productUrl` | `Optional<String>` | yes | Populated after `AdJobCreated`. |
| | `profile` | `Optional<ProductProfile>` | yes | Populated after `ProductScraped`. |
| | `copy` | `Optional<AdCopy>` | yes | Populated after `CopyDrafted`. |
| | `visual` | `Optional<VisualSpec>` | yes | Populated after `VisualGenerated`. |
| | `adPackage` | `Optional<AdPackage>` | yes | Populated after `AdEvaluated`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `AdEvaluated`. |
| | `status` | `AdJobStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AdJobCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `scrapingRejections` | `List<ScrapingRejection>` | no | Appended on every `ScrapingRejected` event; empty on the happy path. |
| | `brandSafetyRejections` | `List<BrandSafetyRejection>` | no | Appended on every `BrandSafetyRejected` event; empty on the happy path. |

Every nullable lifecycle field on `AdJobRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AdJobStatus`: `CREATED`, `SCRAPING`, `SCRAPED`, `COPYING`, `COPIED`, `GENERATING_VISUAL`, `VISUAL_GENERATED`, `EVALUATED`, `FAILED`.

`AdPhase` (used by `@FunctionTool` annotations and `ScrapingPolicyGuardrail`): `SCRAPE`, `COPY`, `VISUAL`.

## Events (`AdJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AdJobCreated` | `productUrl: String` | → CREATED |
| `ScrapeStarted` | — | → SCRAPING |
| `ProductScraped` | `profile: ProductProfile` | → SCRAPED |
| `CopyStarted` | — | → COPYING |
| `CopyDrafted` | `copy: AdCopy` | → COPIED |
| `VisualStarted` | — | → GENERATING_VISUAL |
| `VisualGenerated` | `visual: VisualSpec` | → VISUAL_GENERATED |
| `AdEvaluated` | `eval: EvalResult, adPackage: AdPackage` | → EVALUATED (terminal happy) |
| `ScrapingRejected` | `url: String, reason: String, rejectedAt: Instant` | no status change (audit-only) |
| `BrandSafetyRejected` | `violations: List<BrandViolation>, rejectedAt: Instant` | no status change (audit-only) |
| `AdJobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AdJobRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `scrapingRejections = List.of()`, `brandSafetyRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AdJobRow` mirrors `AdJobRecord` exactly. The UI fetches the full row via `GET /api/ad-jobs/{id}` and streams updates via `GET /api/ad-jobs/sse`.

The view declares ONE query: `getAllAdJobs: SELECT * AS adJobs FROM ad_job_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AdTasks.java`)

```java
public final class AdTasks {
  public static final Task<ProductProfile> SCRAPE_PRODUCT = Task
      .name("Scrape product")
      .description("Fetch the product page and extract a structured ProductProfile")
      .resultConformsTo(ProductProfile.class);

  public static final Task<AdCopy> DRAFT_COPY = Task
      .name("Draft copy")
      .description("Write ad copy variants for each target format from the ProductProfile")
      .resultConformsTo(AdCopy.class);

  public static final Task<VisualSpec> GENERATE_VISUAL = Task
      .name("Generate visual")
      .description("Build an image-generation prompt and select aspect ratio from the AdCopy and ProductProfile")
      .resultConformsTo(VisualSpec.class);

  private AdTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ScrapeTools`, `CopyTools`, and `VisualTools` carries an `AdPhase` constant. `ScrapingPolicyGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G2). For SCRAPE-phase tools it additionally validates the target URL against the allow-list. The tool registry is built once at startup; the guardrail reads it for every call.

## AdQualityScorer — four checks

The deterministic scorer applies four checks on a base score of 1 (one additional point per check satisfied):

1. **Attribute grounding** — every `AdVariant.body` in the `AdCopy.variants` list references at least one `ProductAttribute.name` from the `ProductProfile.attributes` list.
2. **Call-to-action present** — `AdCopy.callToAction` is non-blank.
3. **Visual-copy alignment** — `VisualSpec.imagePrompt` contains the `ProductProfile.productName` string.
4. **Variant coverage** — `AdCopy.variants.size() >= 2` (three expected; two is the minimum to score this point).

Score range: 1–5. Rationale sentence names the first check that failed, or "Attribute grounding, call-to-action presence, visual-copy alignment, and variant coverage all satisfied." when all four pass.
