# Data model — financial-advisor-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MarketDataPoint` | `sector` | `String` | no | Asset class or market segment (e.g., `equities`, `fixed-income`, `alternatives`). |
| | `metric` | `String` | no | Name of the market metric (e.g., `S&P-500-1yr-return`, `10yr-treasury-yield`). |
| | `value` | `String` | no | Metric value as a string (percentage, basis points, etc.). |
| | `source` | `String` | no | In-process data file that provided this reading. |
| | `capturedAt` | `Instant` | no | When the RESEARCH phase recorded this data point. |
| `MarketSnapshot` | `dataPoints` | `List<MarketDataPoint>` | no | Possibly empty (J5 demonstrates the empty path). |
| | `benchmarkIndex` | `String` | no | Name of the reference benchmark (e.g., `S&P 500`). |
| | `benchmarkReturn` | `double` | no | Benchmark 1-year return as a percentage. |
| | `researchedAt` | `Instant` | no | When the RESEARCH task returned. |
| `AllocationTarget` | `assetClass` | `String` | no | Asset class label. |
| | `targetPct` | `double` | no | Target portfolio percentage; all targets in a `Strategy` MUST sum to 100. |
| | `rationale` | `String` | no | One sentence drawn from a `MarketDataPoint` metric. |
| `RationaleItem` | `itemId` | `String` | no | Short stable id (`ri-<sequence>`). |
| | `text` | `String` | no | Rationale text; subject to `SectorSanitizer` redaction. |
| | `supportingMetric` | `String` | no | MUST equal a `metric` field from a `MarketDataPoint` in the upstream `MarketSnapshot`. |
| `Strategy` | `approach` | `String` | no | One-sentence description of the strategic direction. |
| | `allocations` | `List<AllocationTarget>` | no | Possibly empty (J5 empty path). Compliance requires ≥ 2 entries for full score. |
| | `rationale` | `List<RationaleItem>` | no | Possibly empty. |
| | `definedAt` | `Instant` | no | When the STRATEGY task returned. |
| `Instrument` | `ticker` | `String` | no | Exchange ticker symbol. |
| | `name` | `String` | no | Full instrument name. |
| | `assetClass` | `String` | no | Asset class this instrument represents. |
| `ActionItem` | `sequence` | `int` | no | 1-based ordering. |
| | `description` | `String` | no | Action description. Subject to `SectorSanitizer` redaction. |
| | `timing` | `String` | no | Implementation timeframe (e.g., `within 30 days`). |
| | `instruments` | `List<Instrument>` | no | Instruments targeted by this action. |
| `ExecutionPlan` | `actions` | `List<ActionItem>` | no | Possibly empty. |
| | `horizon` | `String` | no | Overall investment horizon (e.g., `3–6 months`). |
| | `plannedAt` | `Instant` | no | When the EXECUTE task returned. |
| `VolatilityMetrics` | `portfolioStdDev` | `double` | no | Annualised portfolio standard deviation. Thresholds: < 0.08 → CONSERVATIVE, < 0.15 → MODERATE, ≥ 0.15 → AGGRESSIVE. |
| | `beta` | `double` | no | Portfolio beta relative to the benchmark. |
| | `sharpeRatio` | `double` | no | Sharpe ratio. |
| `RiskProfile` | `band` | `RiskBand` | no | `CONSERVATIVE`, `MODERATE`, or `AGGRESSIVE`. |
| | `volatility` | `VolatilityMetrics` | no | Computed volatility measures. |
| | `mitigations` | `List<String>` | no | At least 2 entries required for full compliance score. |
| | `assessedAt` | `Instant` | no | When the ASSESS task returned. |
| `FinancialReport` | `title` | `String` | no | 1-line title. |
| | `summary` | `String` | no | 1-paragraph summary. |
| | `strategy` | `Strategy` | no | Embedded from the STRATEGY phase. |
| | `executionPlan` | `ExecutionPlan` | no | Embedded from the EXECUTE phase. |
| | `riskProfile` | `RiskProfile` | no | Embedded from the ASSESS phase. |
| | `reportedAt` | `Instant` | no | When the workflow assembled the report. |
| `ComplianceResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `ComplianceScorer` finished. |
| `DisclaimerInjection` | `advisoryId` | `String` | no | The advisory this injection belongs to. |
| | `disclaimerText` | `String` | no | Full text of the injected disclaimer. |
| | `injectedAt` | `Instant` | no | When `DisclaimerGuardrail` fired. |
| `SanitizerEvent` | `advisoryId` | `String` | no | The advisory this redaction belongs to. |
| | `matchedPattern` | `String` | no | The pattern class that matched (`guaranteed-return`, `directive-securities`, `unlicensed-tax`). |
| | `redactedFragment` | `String` | no | The original text fragment that was replaced. |
| | `firedAt` | `Instant` | no | When `SectorSanitizer` fired. |
| `AdvisoryRecord` (entity state) | `advisoryId` | `String` | no | — |
| | `query` | `Optional<String>` | yes | Populated after `AdvisoryCreated`. |
| | `snapshot` | `Optional<MarketSnapshot>` | yes | Populated after `MarketResearched`. |
| | `strategy` | `Optional<Strategy>` | yes | Populated after `StrategyDefined`. |
| | `executionPlan` | `Optional<ExecutionPlan>` | yes | Populated after `ExecutionPlanned`. |
| | `riskProfile` | `Optional<RiskProfile>` | yes | Populated after `RiskAssessed`. |
| | `report` | `Optional<FinancialReport>` | yes | Populated after `ReportAssembled`. |
| | `compliance` | `Optional<ComplianceResult>` | yes | Populated after `ComplianceScored`. |
| | `status` | `AdvisoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AdvisoryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `disclaimerLog` | `List<DisclaimerInjection>` | no | Appended on every `DisclaimerInjected` event; 4 entries on the happy path. |
| | `sanitizerLog` | `List<SanitizerEvent>` | no | Appended on every `SanitizerFired` event; empty on the clean path. |

Every nullable lifecycle field on `AdvisoryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AdvisoryStatus`: `CREATED`, `RESEARCHING`, `RESEARCHED`, `STRATEGIZING`, `STRATEGIZED`, `PLANNING`, `PLANNED`, `ASSESSING`, `EVALUATED`, `FAILED`.

`RiskBand`: `CONSERVATIVE`, `MODERATE`, `AGGRESSIVE`.

`AdvisorPhase` (used by the function-tool registry and governance hooks): `RESEARCH`, `STRATEGY`, `EXECUTE`, `ASSESS`.

## Events (`AdvisoryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AdvisoryCreated` | `query: String` | → CREATED |
| `ResearchStarted` | — | → RESEARCHING |
| `MarketResearched` | `snapshot: MarketSnapshot` | → RESEARCHED |
| `StrategyStarted` | — | → STRATEGIZING |
| `StrategyDefined` | `strategy: Strategy` | → STRATEGIZED |
| `ExecutionStarted` | — | → PLANNING |
| `ExecutionPlanned` | `executionPlan: ExecutionPlan` | → PLANNED |
| `RiskStarted` | — | → ASSESSING |
| `RiskAssessed` | `riskProfile: RiskProfile` | no status change (sub-step in riskStep) |
| `ReportAssembled` | `report: FinancialReport` | no status change (sub-step in riskStep) |
| `ComplianceScored` | `compliance: ComplianceResult` | → EVALUATED (terminal happy) |
| `DisclaimerInjected` | `advisoryId, disclaimerText, injectedAt` | no status change (audit side-event) |
| `SanitizerFired` | `matchedPattern, redactedFragment, firedAt` | no status change (audit side-event) |
| `AdvisoryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AdvisoryRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `disclaimerLog = List.of()`, `sanitizerLog = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AdvisoryRow` mirrors `AdvisoryRecord` exactly. The UI fetches the full row via `GET /api/advisories/{id}` and streams updates via `GET /api/advisories/sse`.

The view declares ONE query: `getAllAdvisories: SELECT * AS advisories FROM advisory_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AdvisorTasks.java`)

```java
public final class AdvisorTasks {
  public static final Task<MarketSnapshot> RESEARCH_MARKET = Task
      .name("Research market")
      .description("Gather market data points relevant to the query by calling fetchMarketData and lookupBenchmark")
      .resultConformsTo(MarketSnapshot.class);

  public static final Task<Strategy> DEFINE_STRATEGY = Task
      .name("Define strategy")
      .description("Build allocation targets and rationale items from the market snapshot")
      .resultConformsTo(Strategy.class);

  public static final Task<ExecutionPlan> PLAN_EXECUTION = Task
      .name("Plan execution")
      .description("Sequence action items and resolve instruments from the strategy")
      .resultConformsTo(ExecutionPlan.class);

  public static final Task<RiskProfile> ASSESS_RISK = Task
      .name("Assess risk")
      .description("Measure volatility and classify the risk band for the execution plan")
      .resultConformsTo(RiskProfile.class);

  private AdvisorTasks() {}
}
```

The companion Tasks class is mandatory for any `AutonomousAgent` (Lesson 7).

## Compliance scoring (`ComplianceScorer`)

Four checks, one point per check satisfied, on a base of 1:

1. **Risk-band present** — `RiskProfile.band` is non-null.
2. **Mitigations listed** — `RiskProfile.mitigations.size() >= 2`.
3. **Allocation breadth** — `Strategy.allocations.size() >= 2`.
4. **Source traceability** — every `RationaleItem.supportingMetric` in the `Strategy` matches a `metric` field from the `MarketSnapshot.dataPoints`.

Score range: 1–5. Rationale is a single sentence naming the largest gap (or "all checks passed" on a score of 5). A score ≤ 2 causes the UI to highlight the advisory card's border red.
