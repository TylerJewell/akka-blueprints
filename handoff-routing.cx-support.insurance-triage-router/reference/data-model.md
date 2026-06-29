# Data model — auto-insurance-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingRequest` | `requestId` | `String` | no | Server-assigned UUID. |
| | `memberId` | `String` | no | Opaque member identifier from the inbound channel. |
| | `channel` | `String` | no | `"phone"` / `"mobile-app"` / `"web-portal"` / `"email"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedRequest` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["member-id","policy-number","vin","drivers-license","phone","account-number"]`. |
| `TriageDecision` | `category` | `RequestCategory` | no | `CLAIM` / `POLICY` / `REWARDS` / `ROADSIDE` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `MemberResponse` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 2–5 paragraphs. |
| | `action` | `ResponseAction` | no | See enum below. |
| | `specialistTag` | `String` | no | `"claims"` / `"policy"` / `"rewards"` / `"roadside"`. |
| | `confirmationRef` | `Optional<String>` | yes | Roadside dispatch reference (format `RSA-XXXXXXXX`); `Optional.empty()` for all other specialists. |
| | `resolvedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `TriageScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `MemberRequest` (entity state) | `requestId` | `String` | no | — |
| | `incoming` | `IncomingRequest` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedRequest>` | yes | Populated after `RequestSanitized`. |
| | `triage` | `Optional<TriageDecision>` | yes | Populated after `TriageDecided`. |
| | `response` | `Optional<MemberResponse>` | yes | Populated after `ResponseDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `triageScore` | `Optional<TriageScore>` | yes | Populated after `TriageScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `RequestEscalated`. |
| | `status` | `RequestStatus` | no | See enum below. |
| | `createdAt` | `Instant` | no | When `RequestRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `MemberRequest` record is reused as the `RequestView` row type.

## Enums

`RequestCategory`: `CLAIM`, `POLICY`, `REWARDS`, `ROADSIDE`, `UNCLEAR`.

`ResponseAction`: `CLAIM_ACKNOWLEDGED`, `CLAIM_STATUS_PROVIDED`, `POLICY_INFO_PROVIDED`, `COVERAGE_CONFIRMED`, `REWARDS_REDEEMED`, `REWARDS_INFO_PROVIDED`, `ROADSIDE_DISPATCHED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`RequestStatus`: `RECEIVED`, `SANITIZED`, `TRIAGED`, `ROUTED_CLAIM`, `ROUTED_POLICY`, `ROUTED_REWARDS`, `ROUTED_ROADSIDE`, `RESPONSE_DRAFTED`, `BLOCKED`, `RESOLVED`, `ESCALATED`.

## Events (`RequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RequestRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `RequestSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `TriageDecided` | `triage` | `InsuranceWorkflow.triageStep` | → `TRIAGED` |
| `RequestRouted` | `category` | `InsuranceWorkflow.routeStep` | → `ROUTED_CLAIM` / `ROUTED_POLICY` / `ROUTED_REWARDS` / `ROUTED_ROADSIDE` |
| `ResponseDrafted` | `response` | `InsuranceWorkflow.<specialist>Step` | → `RESPONSE_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `InsuranceWorkflow.guardrailStep` | (no status change) |
| `ResponsePublished` | — | `InsuranceWorkflow.publishStep` (guardrail allowed) | → `RESOLVED` (terminal) |
| `ResponseBlocked` | `violations` | `InsuranceWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `RequestEscalated` | `escalationReason` | `InsuranceWorkflow.escalateStep` (triage UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `TriageScored` | `score, rationale, scoredAt` | `TriageEvalScorer` Consumer | (no status change; populates `triageScore`) |

A `BLOCKED → RESOLVED` transition is emitted via `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ResponsePublished` in the same command.

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `InboundRequestReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`RequestRow` is `MemberRequest` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS requests FROM request_view` with no `WHERE category` or `WHERE status` filter — callers (`InsuranceEndpoint`) apply those client-side.
