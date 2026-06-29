# Data model — pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Signal` | `source` | `String` | no | Short name of the originating source. |
| | `url` | `String` | no | Canonical url of the source entry. |
| | `snippet` | `String` | no | Quoted passage retrieved by the COLLECT phase. |
| | `capturedAt` | `Instant` | no | When the COLLECT phase recorded the signal. |
| `SignalSet` | `signals` | `List<Signal>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `collectedAt` | `Instant` | no | When the COLLECT task returned. |
| `Claim` | `claimId` | `String` | no | Short stable id (`c-<8 hex>`). |
| | `text` | `String` | no | The claim, paraphrased from the snippet. |
| | `supportingSource` | `String` | no | MUST equal a `Signal.source` from the upstream `SignalSet`. |
| `Theme` | `themeId` | `String` | no | Slug. MUST be unique within an `Analysis`. |
| | `label` | `String` | no | Human-readable theme name. |
| | `claimIds` | `List<String>` | no | Each entry MUST equal a `Claim.claimId` from the same `Analysis`. |
| `Analysis` | `themes` | `List<Theme>` | no | Possibly empty. |
| | `claims` | `List<Claim>` | no | Possibly empty. |
| | `analyzedAt` | `Instant` | no | When the ANALYZE task returned. |
| `Source` | `label` | `String` | no | Display label (typically the signal source name). |
| | `url` | `String` | no | MUST equal a `Signal.url` from the upstream `SignalSet`. |
| `Section` | `themeId` | `String` | no | MUST equal a `Theme.themeId` from the upstream `Analysis`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–4 sentences paraphrasing the theme's claims. |
| | `sources` | `List<Source>` | no | Non-empty (E1 rule 2). |
| `Briefing` | `title` | `String` | no | 1-line title. |
| | `summary` | `String` | no | 1-paragraph summary. |
| | `sections` | `List<Section>` | no | `sections.size() == analysis.themes.size()` (E1 rule 4). |
| | `writtenAt` | `Instant` | no | When the REPORT task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `CompletenessScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `COLLECT` / `ANALYZE` / `REPORT`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `PhaseGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `BriefingRecord` (entity state) | `briefingId` | `String` | no | — |
| | `topic` | `Optional<String>` | yes | Populated after `BriefingCreated`. |
| | `signals` | `Optional<SignalSet>` | yes | Populated after `SignalsCollected`. |
| | `analysis` | `Optional<Analysis>` | yes | Populated after `AnalysisProduced`. |
| | `briefing` | `Optional<Briefing>` | yes | Populated after `ReportWritten`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `BriefingStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `BriefingCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `BriefingRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BriefingStatus`: `CREATED`, `COLLECTING`, `COLLECTED`, `ANALYZING`, `ANALYZED`, `REPORTING`, `REPORTED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `PhaseGuardrail`): `COLLECT`, `ANALYZE`, `REPORT`.

## Events (`BriefingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BriefingCreated` | `topic: String` | → CREATED |
| `CollectStarted` | — | → COLLECTING |
| `SignalsCollected` | `signals: SignalSet` | → COLLECTED |
| `AnalyzeStarted` | — | → ANALYZING |
| `AnalysisProduced` | `analysis: Analysis` | → ANALYZED |
| `ReportStarted` | — | → REPORTING |
| `ReportWritten` | `briefing: Briefing` | → REPORTED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `BriefingFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BriefingRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BriefingRow` mirrors `BriefingRecord` exactly. The UI fetches the full row via `GET /api/briefings/{id}` and streams updates via `GET /api/briefings/sse`.

The view declares ONE query: `getAllBriefings: SELECT * AS briefings FROM briefing_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`ReportTasks.java`)

```java
public final class ReportTasks {
  public static final Task<SignalSet> COLLECT_SIGNALS = Task
      .name("Collect signals")
      .description("Gather raw signals about a topic by calling searchSignals and fetchSnippet")
      .resultConformsTo(SignalSet.class);

  public static final Task<Analysis> ANALYZE_SIGNALS = Task
      .name("Analyze signals")
      .description("Extract claims, then cluster claims into themes")
      .resultConformsTo(Analysis.class);

  public static final Task<Briefing> WRITE_REPORT = Task
      .name("Write report")
      .description("Compose a Briefing whose Sections mirror the Analysis themes one-to-one")
      .resultConformsTo(Briefing.class);

  private ReportTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `CollectTools`, `AnalyzeTools`, and `ReportTools` carries a `Phase` constant. `PhaseGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.
