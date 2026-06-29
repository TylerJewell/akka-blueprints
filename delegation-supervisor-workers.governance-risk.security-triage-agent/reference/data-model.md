# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## Incident (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `incidentId` | `String` | no | UUID, also the workflow id |
| `assetId` | `String` | no | Identifier of the affected asset |
| `assetType` | `String` | no | Asset class (e.g., "api-gateway", "database") |
| `signalDescription` | `String` | no | Human-readable description of the incident signal |
| `reportedBy` | `String` | no | Identity of the reporter (operator or simulator) |
| `status` | `IncidentStatus` | no | Lifecycle state |
| `severity` | `Severity` | no | Initial severity supplied by the reporter |
| `vulnerabilities` | `Optional<VulnerabilityBundle>` | yes | VulnerabilityScanner output; null until `VulnerabilitiesAttached` |
| `threatContext` | `Optional<ThreatContextBundle>` | yes | ThreatContextAgent output; null until `ThreatContextAttached` |
| `triageReport` | `Optional<TriageReport>` | yes | Merged triage report; null until `TriageReportReady` |
| `approval` | `Optional<ApprovalDecision>` | yes | Officer's decision; null until `IncidentMitigated` or `IncidentRejected` |
| `failureReason` | `Optional<String>` | yes | Set on `IncidentDegraded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `TriageEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `receivedAt` | `Instant` | no | Incident creation time |
| `resolvedAt` | `Optional<Instant>` | yes | Set on any terminal transition (MITIGATED, REJECTED, DEGRADED) |

The `IncidentView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record IncidentSignal(String assetId, String assetType, String signalDescription,
                      String reportedBy, Severity initialSeverity) {}

record ScanPlan(String vulnerabilityScanQuery, String threatContextQuery) {}

record VulnerabilityFinding(String cveId, String description, double cvssScore,
                            String affectedComponent, String source) {}

record VulnerabilityBundle(List<VulnerabilityFinding> vulnerabilities,
                           Severity aggregateSeverity, Instant scannedAt) {}

record ThreatActorContext(String actorGroup, String attackPattern, String targetProfile,
                         String historicalPrecedent, Instant contextAt) {}

record ThreatContextBundle(List<ThreatActorContext> actors, String summary, Instant gatheredAt) {}

record TriageReport(String summary, VulnerabilityBundle vulnerabilities,
                    ThreatContextBundle threatContext, RiskLevel riskLevel,
                    String mitigationPlan, Instant triageAt) {}

record ApprovalDecision(String officerId, String decision, Optional<String> reason,
                        Instant decidedAt) {}
```

## Status enums

```java
enum IncidentStatus { RECEIVED, TRIAGING, AWAITING_APPROVAL, MITIGATED, REJECTED, DEGRADED }
enum Severity { LOW, MEDIUM, HIGH, CRITICAL }
enum RiskLevel { LOW, MODERATE, HIGH, CRITICAL }
```

## Events

### IncidentEntity

| Event | Trigger |
|---|---|
| `IncidentReceived` | Workflow creates the incident (`receiveIncident`) |
| `TriagingStarted` | Workflow begins the parallel scan phase |
| `VulnerabilitiesAttached` | VulnerabilityScanner returns a `VulnerabilityBundle` |
| `ThreatContextAttached` | ThreatContextAgent returns a `ThreatContextBundle` |
| `TriageReportReady` | SecurityCoordinator triage passes; report attached |
| `AwaitingApproval` | Workflow parks; ApprovalEntity notified |
| `IncidentMitigated` | Officer approved; mitigation plan executed |
| `IncidentRejected` | Officer rejected; reason recorded |
| `IncidentDegraded` | A worker timed out; triaged from partial input |
| `TriageEvalScored` | `EvalSampler` recorded a 1–5 quality score |

### IncidentQueue

| Event | Trigger |
|---|---|
| `IncidentSubmitted` | `submitIncident(assetId, signalDescription, reportedBy, initialSeverity)` |

Fields: `{ incidentId, assetId, signalDescription, reportedBy, submittedAt }`.

### ApprovalEntity

| Event | Trigger |
|---|---|
| `ApprovalRequested` | Workflow calls `requestApproval(incidentId)` |
| `ApprovalGranted` | Officer calls `POST /api/incidents/{id}/approve` |
| `ApprovalDenied` | Officer calls `POST /api/incidents/{id}/reject` |
