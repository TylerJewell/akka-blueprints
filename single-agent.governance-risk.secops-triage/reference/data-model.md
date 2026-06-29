# Data model — secops-triage

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RawFinding` | `cveId` | `String` | no | CVE identifier (e.g., `CVE-2024-12345`). |
| | `cvssScore` | `double` | no | CVSS base score (0.0–10.0). |
| | `affectedAsset` | `String` | no | Asset identifier from the submitting scanner. |
| | `description` | `String` | no | Vulnerability description from the scanner. |
| | `submittedBy` | `String` | no | Scanner or user identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `AssetContext` | `assetId` | `String` | no | Matches `RawFinding.affectedAsset`. |
| | `assetCriticality` | `String` | no | `TIER1`, `TIER2`, or `TIER3`. |
| | `internetFacing` | `boolean` | no | Whether the asset is directly reachable from the internet. |
| | `existingMitigations` | `String` | no | Description of compensating controls in place, or `"None"`. |
| | `ownerTeam` | `String` | no | Team responsible for the asset. |
| `EnrichedFinding` | `raw` | `RawFinding` | no | Original submitted finding. |
| | `assetContext` | `AssetContext` | no | Looked up from in-process asset registry. |
| | `threatIntelSummary` | `String` | no | Narrative on exploit availability and wild sightings. |
| | `exploitInWild` | `boolean` | no | Whether active exploitation has been observed. |
| | `enrichedAt` | `Instant` | no | When `FindingEnricher` finished. |
| `TriageVerdict` | `priority` | `TriagePriority` | no | Enum value. |
| | `riskRationale` | `String` | no | 2–4 sentences. |
| | `recommendedAction` | `String` | no | Actionable sentence beginning with a verb. |
| | `requiresApproval` | `boolean` | no | True for `CRITICAL_IMMEDIATE` and `HIGH_SCHEDULED`. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `ApprovalDecision` | `findingId` | `String` | no | Matches `Finding.findingId`. |
| | `status` | `ApprovalStatus` | no | Enum value. |
| | `analystId` | `String` | no | Analyst identifier. |
| | `reason` | `String` | no | Free-text rationale for approve or reject. |
| | `decidedAt` | `Instant` | no | When the analyst acted. |
| `DriftAlert` | `alertId` | `String` | no | UUID minted by `DriftEvaluator`. |
| | `description` | `String` | no | Human-readable drift narrative. |
| | `baselineCriticalPct` | `double` | no | Baseline `CRITICAL_IMMEDIATE` percentage. |
| | `observedCriticalPct` | `double` | no | Observed `CRITICAL_IMMEDIATE` percentage in last 50 findings. |
| | `detectedAt` | `Instant` | no | When `DriftEvaluator` computed the alert. |
| `Finding` (entity state) | `findingId` | `String` | no | UUID minted by `FindingEndpoint`. |
| | `raw` | `Optional<RawFinding>` | yes | Populated after `FindingIngested`. |
| | `enriched` | `Optional<EnrichedFinding>` | yes | Populated after `FindingEnriched`. |
| | `verdict` | `Optional<TriageVerdict>` | yes | Populated after `VerdictRecorded`. |
| | `approval` | `Optional<ApprovalDecision>` | yes | Populated after `RemediationApproved` or `RemediationRejected`. |
| | `driftAlert` | `Optional<DriftAlert>` | yes | Populated after `DriftAlertRaised`. |
| | `status` | `FindingStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `FindingIngested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Finding` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TriagePriority`: `CRITICAL_IMMEDIATE`, `HIGH_SCHEDULED`, `MEDIUM_MONITORED`, `LOW_ACCEPTED`.
`ApprovalStatus`: `PENDING`, `GRANTED`, `REJECTED`.
`FindingStatus`: `INGESTED`, `ENRICHED`, `TRIAGING`, `VERDICT_RECORDED`, `PENDING_APPROVAL`, `REMEDIATED`, `REMEDIATION_REJECTED`, `MONITORED`, `ACCEPTED`, `FAILED`.

## Events (`FindingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `FindingIngested` | `raw` | → INGESTED |
| `FindingEnriched` | `enriched` | → ENRICHED |
| `TriageStarted` | — | → TRIAGING |
| `VerdictRecorded` | `verdict` | → VERDICT_RECORDED |
| `ApprovalRequested` | — | → PENDING_APPROVAL |
| `RemediationApproved` | `approval` | → REMEDIATED (terminal happy) |
| `RemediationRejected` | `approval` | → REMEDIATION_REJECTED (terminal) |
| `FindingMonitored` | — | → MONITORED (terminal, medium) |
| `FindingAccepted` | — | → ACCEPTED (terminal, low) |
| `DriftAlertRaised` | `alert` | → (on sentinel entity only; no state change) |
| `FindingFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Finding.initial("")` with all `Optional` fields as `Optional.empty()` and `status = INGESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## Events (`ApprovalEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ApprovalCreated` | `findingId` | → PENDING |
| `ApprovalGranted` | `analystId, reason, decidedAt` | → GRANTED |
| `ApprovalRejected` | `analystId, reason, decidedAt` | → REJECTED |

## View row

`FindingRow` mirrors `Finding` exactly (including all `Optional` fields). The raw `description` text from `RawFinding` is retained on the row — it is not sensitive and helps analysts read context without a separate fetch.

The view declares ONE query: `getAllFindings: SELECT * AS findings FROM finding_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`TriageTasks.java`)

```java
public final class TriageTasks {
  public static final Task<TriageVerdict> TRIAGE_FINDING = Task
      .name("Triage finding")
      .description("Read the attached enriched security finding and produce a TriageVerdict with priority, risk rationale, and recommended action")
      .resultConformsTo(TriageVerdict.class);

  private TriageTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
