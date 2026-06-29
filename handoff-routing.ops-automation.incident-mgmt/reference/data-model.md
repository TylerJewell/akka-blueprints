# Data model — incident-management

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundReport` | `incidentId` | `String` | no | Server-assigned UUID. |
| | `reportedBy` | `String` | no | Identity of the reporter (user or automation). |
| | `alertSource` | `String` | no | `"pagerduty"` / `"datadog"` / `"manual"`. |
| | `title` | `String` | no | Short alert title. |
| | `description` | `String` | no | Full alert description; raw, pre-enrichment. |
| | `reportedAt` | `Instant` | no | When the alert was received. |
| `EnrichedReport` | `incidentId` | `String` | no | Matches `InboundReport.incidentId`. |
| | `title` | `String` | no | Passed through from the inbound report. |
| | `description` | `String` | no | Passed through from the inbound report. |
| | `hostGroup` | `String` | no | e.g. `"prod-us-east-1"` / `"staging-eu-west-1"`. |
| | `serviceTier` | `String` | no | `"critical"` / `"standard"` / `"dev"`. |
| | `tags` | `List<String>` | no | e.g. `["network","latency","cpu","deploy"]`. |
| `ClassificationDecision` | `category` | `IncidentCategory` | no | `INFRASTRUCTURE` / `APPLICATION` / `AMBIGUOUS`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `RemediationPlan` | `summary` | `String` | no | Two sentences max. |
| | `details` | `String` | no | Full step-by-step for the on-call engineer. |
| | `action` | `RemediationAction` | no | `RESTART_SERVICE` / `SCALE_DEPLOYMENT` / `ROLLBACK_DEPLOY` / `OPEN_CHANGE_RECORD` / `PAGE_ON_CALL` / `INFO_PROVIDED`. |
| | `specialistTag` | `String` | no | `"infra"` / `"app"`. |
| | `proposedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Incident` (entity state) | `incidentId` | `String` | no | — |
| | `report` | `InboundReport` | no | Captured once at create. |
| | `enriched` | `Optional<EnrichedReport>` | yes | Populated after `IncidentEnriched`. |
| | `classification` | `Optional<ClassificationDecision>` | yes | Populated after `IncidentClassified`. |
| | `remediation` | `Optional<RemediationPlan>` | yes | Populated after `PlanDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `IncidentEscalated`. |
| | `status` | `IncidentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `IncidentRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record.

## Enums

`IncidentCategory`: `INFRASTRUCTURE`, `APPLICATION`, `AMBIGUOUS`.

`RemediationAction`: `RESTART_SERVICE`, `SCALE_DEPLOYMENT`, `ROLLBACK_DEPLOY`, `OPEN_CHANGE_RECORD`, `PAGE_ON_CALL`, `INFO_PROVIDED`.

`IncidentStatus`: `RECEIVED`, `ENRICHED`, `CLASSIFIED`, `ROUTED_INFRA`, `ROUTED_APP`, `PLAN_DRAFTED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`IncidentEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `IncidentRegistered` | `report` | `ContextEnricher.registerReport` | → `RECEIVED` |
| `IncidentEnriched` | `enriched` | `ContextEnricher.attachEnriched` | → `ENRICHED` |
| `IncidentClassified` | `classification` | `IncidentWorkflow.classifyStep` | → `CLASSIFIED` |
| `IncidentRouted` | `category` | `IncidentWorkflow.routeStep` | → `ROUTED_INFRA` or `ROUTED_APP` |
| `PlanDrafted` | `remediation` | `IncidentWorkflow.infraStep` / `appStep` | → `PLAN_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `IncidentWorkflow.guardrailStep` | (no status change) |
| `RemediationPublished` | — | `IncidentWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `RemediationBlocked` | `violations` | `IncidentWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `IncidentEscalated` | `escalationReason` | `IncidentWorkflow.escalateStep` (category AMBIGUOUS or unrecoverable error) | → `ESCALATED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `BLOCKED → RESOLVED` transition is emitted via `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `RemediationPublished` in the same command.

## Events (`IncidentQueue`)

| Event | Payload |
|---|---|
| `InboundReportReceived` | `report` (the raw, pre-enrichment payload — used as the audit log) |

## View row

`IncidentRow` is `Incident` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS incidents FROM incident_view` with no `WHERE category` or `WHERE status` filter — callers (`IncidentEndpoint`) apply those client-side.
