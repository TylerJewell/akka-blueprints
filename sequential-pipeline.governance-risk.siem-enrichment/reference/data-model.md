# Data model — siem-enrichment

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RawAlertField` | `key` | `String` | no | SIEM field name (e.g., `event.action`). |
| | `value` | `String` | no | SIEM field value. |
| `AlertDetail` | `alertId` | `String` | no | Stable id for the alert throughout the pipeline. |
| | `ruleName` | `String` | no | Originating SIEM detection rule name. |
| | `severity` | `String` | no | Raw SIEM severity: `LOW`, `MEDIUM`, `HIGH`, or `CRITICAL`. |
| | `sourceIp` | `String` | no | Source IP address from the SIEM event. |
| | `destinationIp` | `String` | no | Destination IP address. |
| | `hostname` | `String` | no | Hostname associated with the alert. |
| | `rawFields` | `List<RawAlertField>` | no | All key-value pairs from the raw SIEM JSON; possibly empty for minimal alerts. |
| | `detectedAt` | `Instant` | no | When the originating SIEM rule fired. |
| `AttackPattern` | `patternId` | `String` | no | Short stable id (`ap-<8 hex>`). |
| | `techniqueId` | `String` | no | MITRE ATT&CK technique ID (e.g., `T1021.002`). MUST resolve in the bundled registry (E1 rule 2). |
| | `techniqueName` | `String` | no | Human-readable technique name. |
| | `tactic` | `String` | no | ATT&CK tactic (e.g., `Lateral Movement`). |
| | `confidence` | `String` | no | `LOW`, `MEDIUM`, or `HIGH`. |
| | `indicator` | `String` | no | The raw field key-value that triggered the lookup (e.g., `destination.port=445`). |
| `EnrichedAlert` | `alertDetail` | `AlertDetail` | no | The upstream `AlertDetail` carried forward. |
| | `attackPatterns` | `List<AttackPattern>` | no | Possibly empty (J6 demonstrates the empty path). |
| | `derivedSeverity` | `String` | no | Severity derived from the highest-confidence `AttackPattern`; falls back to `AlertDetail.severity` when `attackPatterns` is empty. |
| | `enrichedAt` | `Instant` | no | When the ENRICH task returned. |
| `ZendeskTicketRef` | `ticketId` | `String` | no | Zendesk ticket identifier (e.g., `ZD-10042`). |
| | `ticketUrl` | `String` | no | Zendesk ticket URL. |
| `TriageTicket` | `alertId` | `String` | no | Matches `AlertDetail.alertId`. |
| | `title` | `String` | no | One-line ticket title including severity and technique IDs. |
| | `severity` | `String` | no | MUST equal `EnrichedAlert.derivedSeverity` (E1 rule 3). |
| | `assignedTeam` | `String` | no | Non-empty (E1 rule 4). Determined by the highest-severity tactic. |
| | `attackPatterns` | `List<AttackPattern>` | no | Full list from `EnrichedAlert.attackPatterns`. |
| | `zendeskRef` | `ZendeskTicketRef` | no | Reference returned by `openZendeskTicket`. |
| | `summary` | `String` | no | 2–4 sentences describing the alert, techniques, and routing decision. |
| | `createdAt` | `Instant` | no | When the TRIAGE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `TriageQualityScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `FETCH`, `ENRICH`, or `TRIAGE`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `TicketScopeGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `AlertRecord` (entity state) | `alertId` | `String` | no | — |
| | `rawAlertJson` | `Optional<String>` | yes | Populated after `AlertReceived`. |
| | `alertDetail` | `Optional<AlertDetail>` | yes | Populated after `FetchCompleted`. |
| | `enrichedAlert` | `Optional<EnrichedAlert>` | yes | Populated after `EnrichmentProduced`. |
| | `triageTicket` | `Optional<TriageTicket>` | yes | Populated after `TicketCreated`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `AlertStatus` | no | See enum. |
| | `receivedAt` | `Instant` | no | When `AlertReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `AlertRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AlertStatus`: `RECEIVED`, `FETCHING`, `FETCHED`, `ENRICHING`, `ENRICHED`, `TRIAGING`, `TRIAGED`, `EVALUATED`, `FAILED`.

`Phase` (used by `@FunctionTool` annotations and `TicketScopeGuardrail`): `FETCH`, `ENRICH`, `TRIAGE`.

## Events (`AlertEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AlertReceived` | `rawAlertJson: String` | → RECEIVED |
| `FetchStarted` | — | → FETCHING |
| `FetchCompleted` | `alertDetail: AlertDetail` | → FETCHED |
| `EnrichStarted` | — | → ENRICHING |
| `EnrichmentProduced` | `enrichedAlert: EnrichedAlert` | → ENRICHED |
| `TriageStarted` | — | → TRIAGING |
| `TicketCreated` | `triageTicket: TriageTicket` | → TRIAGED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `AlertFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AlertRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AlertRow` mirrors `AlertRecord` exactly. The UI fetches the full row via `GET /api/alerts/{id}` and streams updates via `GET /api/alerts/sse`.

The view declares ONE query: `getAllAlerts: SELECT * AS alerts FROM alert_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AlertTasks.java`)

```java
public final class AlertTasks {
  public static final Task<AlertDetail> FETCH_ALERT = Task
      .name("Fetch alert")
      .description("Retrieve the raw alert detail and host context using fetchAlertDetail and fetchHostContext")
      .resultConformsTo(AlertDetail.class);

  public static final Task<EnrichedAlert> ENRICH_ALERT = Task
      .name("Enrich alert")
      .description("Map alert indicators to MITRE ATT&CK techniques using lookupMitreTechnique and mapAttackPattern")
      .resultConformsTo(EnrichedAlert.class);

  public static final Task<TriageTicket> CREATE_TICKET = Task
      .name("Create ticket")
      .description("Open a Zendesk triage ticket for the enriched alert using openZendeskTicket and assignTicketTeam")
      .resultConformsTo(TriageTicket.class);

  private AlertTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `FetchTools`, `EnrichTools`, and `TicketTools` carries a `Phase` constant. `TicketScopeGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). The tool registry is built once at startup; the guardrail reads it for every call.
