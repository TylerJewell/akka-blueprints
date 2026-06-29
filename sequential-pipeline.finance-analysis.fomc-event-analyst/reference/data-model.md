# Data model — fomc-event-analyst

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MarketIndicator` | `indicatorId` | `String` | no | Short stable id (e.g. `cpi-yoy-2026-06`). |
| | `name` | `String` | no | Human-readable indicator name. |
| | `value` | `double` | no | Numeric reading. |
| | `unit` | `String` | no | Unit of measure (e.g. `percent`, `index`). |
| | `observedAt` | `Instant` | no | When the data point was published. |
| `YieldCurveSnapshot` | `twoYear` | `double` | no | 2-year Treasury yield, in percent. |
| | `tenYear` | `double` | no | 10-year Treasury yield, in percent. |
| | `spread` | `double` | no | `tenYear - twoYear`. |
| | `snapshotAt` | `Instant` | no | Timestamp of the yield curve reading. |
| `MarketSnapshot` | `eventId` | `String` | no | FOMC event identifier. |
| | `indicators` | `List<MarketIndicator>` | no | Possibly empty when no sample data exists for the event. |
| | `yieldCurve` | `YieldCurveSnapshot` | no | Zero-valued when no data exists. |
| | `gatheredAt` | `Instant` | no | When the GATHER task returned. |
| `PolicySignal` | `signalId` | `String` | no | Short stable id (`ps-<8 hex>`). |
| | `direction` | `String` | no | `"hawkish"`, `"dovish"`, or `"neutral"`. |
| | `magnitude` | `String` | no | `"strong"`, `"moderate"`, or `"weak"`. |
| | `rationale` | `String` | no | 1-sentence rationale. |
| | `groundedIndicatorId` | `String` | no | MUST equal a `MarketIndicator.indicatorId` from the upstream `MarketSnapshot`. |
| `RateMoveForecast` | `forecastId` | `String` | no | Short stable id. |
| | `basisPoints` | `int` | no | Expected rate move: positive = hike, negative = cut, 0 = hold. |
| | `confidence` | `double` | no | 0.0–1.0. |
| | `narrative` | `String` | no | 1-sentence forecast rationale. |
| `PolicySignalSet` | `signals` | `List<PolicySignal>` | no | Possibly empty when the market snapshot had no indicators. |
| | `forecast` | `RateMoveForecast` | no | Always present; `basisPoints = 0` on empty signal set. |
| | `interpretedAt` | `Instant` | no | When the INTERPRET task returned. |
| `GroundingRef` | `indicatorId` | `String` | no | MUST equal a `MarketIndicator.indicatorId` from the upstream `MarketSnapshot`. |
| | `indicatorName` | `String` | no | Display name for the UI. |
| | `value` | `double` | no | The indicator's numeric reading. |
| | `unit` | `String` | no | Unit of measure. |
| `PolicySection` | `signalId` | `String` | no | MUST equal a `PolicySignal.signalId` from the upstream `PolicySignalSet`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–3 sentences paraphrasing the signal's rationale. |
| | `groundedIn` | `List<GroundingRef>` | no | Non-empty (G1 quality check). |
| `PolicyAnalysis` | `eventName` | `String` | no | Human-readable event name (e.g. `"FOMC June 2026 rate decision"`). |
| | `rateOutlook` | `String` | no | `"hike"`, `"cut"`, `"hold"`, or `"hold (no data)"`. |
| | `executiveSummary` | `String` | no | 1–2 sentence summary. |
| | `sections` | `List<PolicySection>` | no | `sections.size() == signals.size()` (G1 signal-attribution check). |
| | `forecast` | `RateMoveForecast` | no | Forwarded from `PolicySignalSet.forecast`. |
| | `synthesizedAt` | `Instant` | no | When the SYNTHESIZE task returned. |
| `ReviewOutcome` | `verdict` | `String` | no | `"ACCEPTED"` or `"REJECTED"`. |
| | `reason` | `String` | no | One sentence naming the result or the first failing check. |
| | `reviewedAt` | `Instant` | no | When `FinancialOutputGuardrail` completed. |
| `GuardrailRejection` | `phase` | `String` | no | `"SYNTHESIZE"` (this guardrail fires only on the SYNTHESIZE task). |
| | `reason` | `String` | no | Structured reason from `FinancialOutputGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `AnalysisRecord` (entity state) | `analysisId` | `String` | no | — |
| | `eventId` | `Optional<String>` | yes | Populated after `AnalysisCreated`. |
| | `snapshot` | `Optional<MarketSnapshot>` | yes | Populated after `MarketDataGathered`. |
| | `signalSet` | `Optional<PolicySignalSet>` | yes | Populated after `PolicySignalsInterpreted`. |
| | `analysis` | `Optional<PolicyAnalysis>` | yes | Populated after `AnalysisSynthesized`. |
| | `review` | `Optional<ReviewOutcome>` | yes | Populated after `OutputReviewed`. |
| | `status` | `AnalysisStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AnalysisCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `AnalysisRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AnalysisStatus`: `CREATED`, `GATHERING`, `GATHERED`, `INTERPRETING`, `INTERPRETED`, `SYNTHESIZING`, `SYNTHESIZED`, `REVIEWED`, `FAILED`.

## Events (`FomcEventEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AnalysisCreated` | `eventId: String` | → CREATED |
| `GatherStarted` | — | → GATHERING |
| `MarketDataGathered` | `snapshot: MarketSnapshot` | → GATHERED |
| `InterpretStarted` | — | → INTERPRETING |
| `PolicySignalsInterpreted` | `signalSet: PolicySignalSet` | → INTERPRETED |
| `SynthesizeStarted` | — | → SYNTHESIZING |
| `AnalysisSynthesized` | `analysis: PolicyAnalysis` | → SYNTHESIZED |
| `OutputReviewed` | `review: ReviewOutcome` | → REVIEWED (terminal happy) |
| `GuardrailRejected` | `phase, reason, rejectedAt` | no status change (audit-only) |
| `AnalysisFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AnalysisRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AnalysisRow` mirrors `AnalysisRecord` exactly. The UI fetches the full row via `GET /api/analyses/{id}` and streams updates via `GET /api/analyses/sse`.

The view declares ONE query: `getAllAnalyses: SELECT * AS analyses FROM analysis_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`FomcTasks.java`)

```java
public final class FomcTasks {
  public static final Task<MarketSnapshot> GATHER_MARKET_DATA = Task
      .name("Gather market data")
      .description("Collect market indicators and yield curve snapshot for an FOMC event using fetchIndicators and fetchYieldCurve")
      .resultConformsTo(MarketSnapshot.class);

  public static final Task<PolicySignalSet> INTERPRET_POLICY_SIGNALS = Task
      .name("Interpret policy signals")
      .description("Extract policy signals from a MarketSnapshot, then classify the rate-move direction and magnitude")
      .resultConformsTo(PolicySignalSet.class);

  public static final Task<PolicyAnalysis> SYNTHESIZE_ANALYSIS = Task
      .name("Synthesize analysis")
      .description("Compose a PolicyAnalysis whose PolicySections mirror the policy signals one-to-one, grounded in the recorded market indicators")
      .resultConformsTo(PolicyAnalysis.class);

  private FomcTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Guardrail quality checks

`FinancialOutputGuardrail` applies three checks in order on every SYNTHESIZE task candidate:

1. **Signal attribution** — every `PolicySection.signalId` must equal a `PolicySignal.signalId` from the recorded `PolicySignalSet.signals`. Failure message: `"signal-attribution: section '<signalId>' not found in PolicySignalSet"`.
2. **Market grounding** — every `PolicySection.groundedIn[i].indicatorId` must equal a `MarketIndicator.indicatorId` from the recorded `MarketSnapshot.indicators`. Failure message: `"market-grounding: section '<signalId>' references indicatorId '<id>' not found in recorded MarketSnapshot"`.
3. **Non-emptiness** — `PolicyAnalysis.sections` must be non-empty, unless `PolicySignalSet.signals` is also empty (vacuous pass on the empty-data path). Failure message: `"non-empty-sections: sections list is empty"`.

On any failure, the guardrail rejects with a prefixed message `"financial-quality-violation: <check>: <detail>"`. The tool registry is built once at startup from the registered tool classes; the guardrail reads the entity state via `analysisId` from the `TaskDef.metadata` for every candidate review.
