# Data model — 311-triage

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundServiceRequest` | `requestId` | `String` | no | Server-assigned UUID. |
| | `constituentId` | `String` | no | Opaque identifier from the inbound channel. |
| | `channel` | `String` | no | `"phone"` / `"web-form"` / `"mobile-app"`. |
| | `subject` | `String` | no | Raw subject; not yet sanitized. |
| | `description` | `String` | no | Raw description body. |
| | `locationHint` | `String` | no | Raw location string; may contain full address. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedServiceRequest` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedDescription` | `String` | no | PII redacted. |
| | `redactedLocation` | `String` | no | Street name kept; unit/apartment redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["name","phone","email","address"]`. |
| `TriageDecision` | `category` | `RequestCategory` | no | `PUBLIC_WORKS` / `PERMITS_ZONING` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `RouteVerdict` | `approved` | `boolean` | no | `true` to proceed; `false` to flag. |
| | `flags` | `List<String>` | no | Empty when `approved=true`. Rubric-token list otherwise. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `DepartmentResponse` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 2–4 paragraphs. |
| | `action` | `DepartmentAction` | no | `WORK_ORDER_CREATED` / `PERMIT_INFO_PROVIDED` / `INSPECTION_SCHEDULED` / `REFERRAL_ISSUED` / `INFO_PROVIDED` / `ESCALATED`. |
| | `departmentTag` | `String` | no | `"public-works"` / `"permits-zoning"`. |
| | `respondedAt` | `Instant` | no | When the specialist finished. |
| `TriageScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `ServiceRequest` (entity state) | `requestId` | `String` | no | — |
| | `inbound` | `InboundServiceRequest` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedServiceRequest>` | yes | Populated after `ServiceRequestSanitized`. |
| | `triage` | `Optional<TriageDecision>` | yes | Populated after `TriageDecided`. |
| | `routeVerdict` | `Optional<RouteVerdict>` | yes | Populated after `RouteVerdictAttached`. |
| | `response` | `Optional<DepartmentResponse>` | yes | Populated after `ResponseDrafted`. |
| | `triageScore` | `Optional<TriageScore>` | yes | Populated after `TriageScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `RequestEscalated`. |
| | `status` | `RequestStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ServiceRequestRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `ServiceRequest` record is reused as the `RequestView` row type.

## Enums

`RequestCategory`: `PUBLIC_WORKS`, `PERMITS_ZONING`, `UNCLEAR`.

`DepartmentAction`: `WORK_ORDER_CREATED`, `PERMIT_INFO_PROVIDED`, `INSPECTION_SCHEDULED`, `REFERRAL_ISSUED`, `INFO_PROVIDED`, `ESCALATED`.

`RequestStatus`: `RECEIVED`, `SANITIZED`, `TRIAGED`, `ROUTE_APPROVED`, `ROUTED_PUBLIC_WORKS`, `ROUTED_PERMITS_ZONING`, `RESPONSE_DRAFTED`, `FLAGGED_FOR_REVIEW`, `RESOLVED`, `ESCALATED`.

## Events (`ServiceRequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ServiceRequestRegistered` | `inbound` | `PiiSanitizer.registerInbound` | → `RECEIVED` |
| `ServiceRequestSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `TriageDecided` | `triage` | `RequestWorkflow.triageStep` | → `TRIAGED` |
| `RouteVerdictAttached` | `routeVerdict` | `RequestWorkflow.routeGuardrailStep` | (no status change by itself) |
| `ServiceRequestRouted` | `category` | `RequestWorkflow.routeStep` | → `ROUTE_APPROVED` then `ROUTED_PUBLIC_WORKS` or `ROUTED_PERMITS_ZONING` |
| `ResponseDrafted` | `response` | `RequestWorkflow.publicWorksStep` / `permitsStep` | → `RESPONSE_DRAFTED` |
| `ResponsePublished` | — | `RequestWorkflow.publishStep` | → `RESOLVED` (terminal) |
| `RequestFlagged` | `flags` | `RequestWorkflow.routeGuardrailStep` (approved=false) | → `FLAGGED_FOR_REVIEW` (holds pending review) |
| `RequestEscalated` | `escalationReason` | `RequestWorkflow.escalateStep` (UNCLEAR or unrecoverable error) or reviewer decision | → `ESCALATED` (terminal) |
| `TriageScored` | `score, rationale, scoredAt` | `TriageEvalScorer` Consumer | (no status change; populates `triageScore`) |

A `FLAGGED_FOR_REVIEW → ROUTE_APPROVED` transition is emitted via the `review` command with `decision="approve"`, which writes `RouteVerdictAttached` (approved=true with the reviewer note) and resumes the workflow. A `FLAGGED_FOR_REVIEW → ESCALATED` transition is emitted via `review` with `decision="escalate"`, which writes `RequestEscalated`.

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `InboundRequestReceived` | `inbound` (the raw, pre-sanitization payload — the audit log) |

## View row

`RequestRow` is `ServiceRequest` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS requests FROM request_view` with no `WHERE category` or `WHERE status` filter — callers (`TriageEndpoint`) apply those client-side.
