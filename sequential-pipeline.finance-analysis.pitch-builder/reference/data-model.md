# Data model — pitch-builder

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RawResearchItem` | `source` | `String` | no | Short name of the originating data source. |
| | `url` | `String` | no | Canonical url of the source entry. |
| | `headline` | `String` | no | Primary headline or title of the research item. |
| | `metadata` | `String` | no | Supplementary metadata field; may contain `[REDACTED]` spans after `PiiSanitizer` runs. |
| | `capturedAt` | `Instant` | no | When the RESEARCH phase recorded the item. |
| `ResearchPack` | `target` | `String` | no | The target company name supplied by the user. |
| | `items` | `List<RawResearchItem>` | no | Possibly empty; J6 demonstrates the empty path. The list contains sanitized items. |
| | `redactionCount` | `int` | no | Total number of PII spans redacted across all items. |
| | `researchedAt` | `Instant` | no | When the RESEARCH task returned and sanitization completed. |
| `PeerCompany` | `ticker` | `String` | no | Exchange-listed ticker symbol. MUST be unique within a `CompsTable`. |
| | `name` | `String` | no | Full company name. |
| | `sector` | `String` | no | GICS sector label (e.g. `"Containers & Packaging"`). |
| | `marketCapM` | `double` | no | Market capitalisation in USD millions at time of comps build. |
| `MultiplesTable` | `evEbitdaLow` | `double` | no | Low end of observed EV/EBITDA range. |
| | `evEbitdaHigh` | `double` | no | High end of observed EV/EBITDA range. |
| | `evRevenueLow` | `double` | no | Low end of observed EV/Revenue range. |
| | `evRevenueHigh` | `double` | no | High end of observed EV/Revenue range. |
| | `peRatioLow` | `double` | no | Low end of observed P/E ratio range. |
| | `peRatioHigh` | `double` | no | High end of observed P/E ratio range. |
| `CompsTable` | `peers` | `List<PeerCompany>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `multiples` | `MultiplesTable` | no | Aggregate ranges across the peer set. |
| | `builtAt` | `Instant` | no | When the COMPARABLES task returned. |
| `CoverPage` | `targetName` | `String` | no | Target company name. |
| | `sector` | `String` | no | Sector derived from `CompsTable.peers` most common sector. |
| | `preparedBy` | `String` | no | Always `"Pitch Builder"`. |
| | `preparedAt` | `Instant` | no | When the DRAFT task produced the cover page. |
| `PitchSection` | `sectionId` | `String` | no | Slug. MUST be unique within a `Pitchbook`. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–4 sentences supporting the section's investment thesis. |
| | `citedTickers` | `List<String>` | no | Each entry MUST appear in `CompsTable.peers[].ticker`. May be empty for non-comps sections. |
| | `citedFigures` | `List<String>` | no | Each string is `"<value>x <type>"`. Numeric component MUST fall within the corresponding `MultiplesTable` range. |
| `Pitchbook` | `cover` | `CoverPage` | no | Cover page metadata. |
| | `executiveSummary` | `String` | no | 2–3-sentence summary. |
| | `sections` | `List<PitchSection>` | no | Non-empty on the happy path; empty if `CompsTable.peers` was empty. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `ValidationResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `validatedAt` | `Instant` | no | When `CitationValidator` finished the scoring pass. |
| `PiiRedactionRecord` | `field` | `String` | no | Name of the `RawResearchItem` field that was redacted. |
| | `patternId` | `String` | no | `personal-name` / `personal-email` / `direct-phone`. |
| | `offsetStart` | `int` | no | Character offset of the start of the redacted span in the original value. |
| | `offsetEnd` | `int` | no | Character offset of the end of the redacted span in the original value. |
| | `loggedAt` | `Instant` | no | When `PiiSanitizer` recorded the redaction. |
| `CitationRejectionRecord` | `sectionId` | `String` | no | ID of the section that failed validation. |
| | `rule` | `String` | no | `unknown-ticker` / `out-of-range-multiple` / `unknown-peer-name`. |
| | `detail` | `String` | no | Full structured reason from `CitationValidator`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `PitchbookRecord` (entity state) | `pitchbookId` | `String` | no | — |
| | `target` | `Optional<String>` | yes | Populated after `PitchbookCreated`. |
| | `research` | `Optional<ResearchPack>` | yes | Populated after `TargetResearched`. Contains sanitized items. |
| | `comps` | `Optional<CompsTable>` | yes | Populated after `ComparablesBuilt`. |
| | `pitchbook` | `Optional<Pitchbook>` | yes | Populated after `DraftWritten`. |
| | `validation` | `Optional<ValidationResult>` | yes | Populated after `ValidationScored`. |
| | `status` | `PitchbookStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PitchbookCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `piiRedactions` | `List<PiiRedactionRecord>` | no | Appended on every `PiiRedactionLogged` event; empty if no PII was found. |
| | `citationRejections` | `List<CitationRejectionRecord>` | no | Appended on every `CitationRejected` event; empty on the happy path. |

Every nullable lifecycle field on `PitchbookRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PitchbookStatus`: `CREATED`, `RESEARCHING`, `RESEARCHED`, `COMPS_RUNNING`, `COMPS_READY`, `DRAFTING`, `DRAFTED`, `VALIDATED`, `FAILED`.

`PitchPhase` (workflow step metadata): `RESEARCH`, `COMPARABLES`, `DRAFT`.

## Events (`PitchbookEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PitchbookCreated` | `target: String` | → CREATED |
| `ResearchStarted` | — | → RESEARCHING |
| `TargetResearched` | `research: ResearchPack` | → RESEARCHED |
| `CompsStarted` | — | → COMPS_RUNNING |
| `ComparablesBuilt` | `comps: CompsTable` | → COMPS_READY |
| `DraftStarted` | — | → DRAFTING |
| `DraftWritten` | `pitchbook: Pitchbook` | → DRAFTED |
| `ValidationScored` | `validation: ValidationResult` | → VALIDATED (terminal happy) |
| `PiiRedactionLogged` | `field, patternId, offsetStart, offsetEnd, loggedAt` | no status change (audit-only) |
| `CitationRejected` | `sectionId, rule, detail, rejectedAt` | no status change (audit-only) |
| `PitchbookFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PitchbookRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `piiRedactions = List.of()`, `citationRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PitchbookRow` mirrors `PitchbookRecord` exactly. The UI fetches the full row via `GET /api/pitchbooks/{id}` and streams updates via `GET /api/pitchbooks/sse`.

The view declares ONE query: `getAllPitchbooks: SELECT * AS pitchbooks FROM pitchbook_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`PitchTasks.java`)

```java
public final class PitchTasks {
  public static final Task<ResearchPack> RESEARCH_TARGET = Task
      .name("Research target")
      .description("Gather raw research items about a target company by calling searchFilings and fetchHeadline")
      .resultConformsTo(ResearchPack.class);

  public static final Task<CompsTable> RUN_COMPARABLES = Task
      .name("Run comparables")
      .description("Select peer companies then build a multiples table from in-process data")
      .resultConformsTo(CompsTable.class);

  public static final Task<Pitchbook> DRAFT_PITCHBOOK = Task
      .name("Draft pitchbook")
      .description("Compose a Pitchbook whose sections cite only peers and multiples from the recorded CompsTable")
      .resultConformsTo(Pitchbook.class);

  private PitchTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## PII sanitizer contracts

`PiiSanitizer.sanitize(ResearchPack raw)` is a pure function: same input always produces the same redacted output (pattern matching is deterministic). It returns a new `ResearchPack` — it never mutates the input record. The original `RawResearchItem` values are discarded after sanitization; only the redacted version is stored on the entity.

`PiiPatternRegistry` exposes three `Pattern` constants: `PERSONAL_NAME`, `PERSONAL_EMAIL`, `DIRECT_PHONE`. Deployers may subclass and override `getPatterns()` to add jurisdiction-specific patterns without changing the sanitizer logic.

## Citation validator contracts

`CitationValidator` has two call paths:
1. **Guardrail path** — called by the `before-agent-response` hook on every `DRAFT_PITCHBOOK` response. Returns `Guardrail.accept()` or `Guardrail.reject(detail)`. Side-effect: calls `PitchbookEntity.logCitationRejection(...)` on reject.
2. **Scoring path** — called directly by `PitchbookPipelineWorkflow.validationStep`. Returns `ValidationResult{score, rationale}`. Four checks, one point each on a base of 1: (1) ticker coverage — every `citedTickers[i]` appears in `CompsTable.peers[].ticker`; (2) figure provenance — every `citedFigures[j]` numeric value is within the matching `MultiplesTable` range; (3) peer name coverage — no peer company name in section body text is absent from `CompsTable.peers[].name`; (4) section count — `sections.size() ≥ 1`.
