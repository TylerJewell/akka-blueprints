# Data model — churn-monitor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `AccountSnapshot` | `accountId` | `String` | no | Unique account identifier from the CRM source. |
| | `accountName` | `String` | no | Raw account name; *not yet sanitized*. |
| | `contactEmail` | `String` | no | Raw primary contact email. |
| | `contactPhone` | `String` | no | Raw primary contact phone. |
| | `industry` | `String` | no | Account industry vertical (e.g., "SaaS", "Retail"). |
| | `accountTier` | `String` | no | Commercial tier (e.g., "enterprise", "mid-market", "smb"). |
| | `monthsActive` | `int` | no | Months since account creation. |
| | `supportTicketsLast90d` | `int` | no | Support tickets opened in the last 90 days. |
| | `usageRatioLast30d` | `double` | no | Product usage as a fraction of licensed capacity, last 30 days. |
| | `snapshotAt` | `Instant` | no | When the CRM snapshot was taken. |
| `SanitizedSnapshot` | `redactedAccountName` | `String` | no | Name tokens replaced with `[REDACTED-NAME]`. |
| | `industry` | `String` | no | Retained as-is (not PII). |
| | `accountTier` | `String` | no | Retained as-is. |
| | `monthsActive` | `int` | no | Retained as-is. |
| | `supportTicketsLast90d` | `int` | no | Retained as-is. |
| | `usageRatioLast30d` | `double` | no | Retained as-is. |
| | `piiCategoriesFound` | `List<String>` | no | e.g., `["account-name","email","phone"]`. |
| `ChurnScore` | `riskLevel` | `RiskLevel` | no | One of `HIGH`, `MEDIUM`, `LOW`. |
| | `probability` | `double` | no | Calibrated 90-day churn probability (0.0–1.0). |
| | `topSignals` | `List<String>` | no | Up to 5 noun phrases naming driving features. |
| `RetentionAction` | `actionType` | `String` | no | One of the five allowed action types. |
| | `description` | `String` | no | Plain-language description of the action. |
| | `priorityRank` | `int` | no | 1 = most urgent. |
| `RetentionPlan` | `actions` | `List<RetentionAction>` | no | 2–4 ranked actions. |
| | `summary` | `String` | no | 1–2 sentence overall approach. |
| | `advisedAt` | `Instant` | no | When the agent finished. |
| `ActionDecision` | `decidedBy` | `String` | no | Manager identifier. |
| | `reason` | `Optional<String>` | yes | Required when dismissing. |
| | `decidedAt` | `Instant` | no | When the manager acted. |
| `EvalSummary` | `driftScore` | `double` | no | 0.0 = no drift; 1.0 = severe drift. |
| | `driftVerdict` | `String` | no | One-sentence finding. |
| | `fairnessScore` | `double` | no | 0.0 = severe bias; 1.0 = no bias. |
| | `fairnessVerdict` | `String` | no | One-sentence finding. |
| | `evaluatedAt` | `Instant` | no | When the eval ran. |
| `ChurnScoreSample` | `accountId` | `String` | no | Account identifier for the batch. |
| | `riskLevel` | `RiskLevel` | no | Scored risk level. |
| | `probability` | `double` | no | Scored probability. |
| | `industry` | `String` | no | Industry vertical for fairness grouping. |
| | `accountTier` | `String` | no | Account tier for fairness grouping. |
| `AccountChurnState` (entity state) | `accountId` | `String` | no | — |
| | `snapshot` | `AccountSnapshot` | no | Captured once at receive. |
| | `sanitized` | `Optional<SanitizedSnapshot>` | yes | Populated after SnapshotSanitized. |
| | `score` | `Optional<ChurnScore>` | yes | Populated after AccountScored. |
| | `retentionPlan` | `Optional<RetentionPlan>` | yes | Populated after RetentionAdvised (HIGH only). |
| | `decision` | `Optional<ActionDecision>` | yes | Populated after ActionRecorded or AccountDismissed. |
| | `latestEval` | `Optional<EvalSummary>` | yes | Most recent EvalCompleted payload. |
| | `status` | `AccountChurnStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When SnapshotReceived was emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

## Enums

`RiskLevel`: `HIGH`, `MEDIUM`, `LOW`.

`AccountChurnStatus`: `RECEIVED`, `SANITIZED`, `SCORED`, `ADVISED`, `AWAITING_ACTION`, `ACTIONED`, `DISMISSED`, `LOW_RISK_CLOSED`, `MEDIUM_RISK_CLOSED`.

## Events (`AccountChurnEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SnapshotReceived` | `snapshot` | → RECEIVED |
| `SnapshotSanitized` | `sanitized` | → SANITIZED |
| `AccountScored` | `score` | → SCORED |
| `RetentionAdvised` | `retentionPlan` | → ADVISED → AWAITING_ACTION (HIGH only) |
| `ActionRecorded` | `decision` | → ACTIONED (terminal) |
| `AccountDismissed` | `decision` | → DISMISSED (terminal) |
| `AccountClosed` | `reason: String` | → LOW_RISK_CLOSED or MEDIUM_RISK_CLOSED (terminal) |
| `EvalCompleted` | `evalSummary` | (no status change; populates latestEval) |

## Events (`AccountSnapshotQueue`)

| Event | Payload |
|---|---|
| `SnapshotReceived` | `snapshot` (raw, pre-sanitization — used by the audit log) |

## View row

`AccountChurnRow` mirrors `AccountChurnState` but omits raw PII fields from `AccountSnapshot` (accountName, contactEmail, contactPhone). The UI fetches the full state on-demand via `GET /api/churn/{accountId}`; the list view uses `ChurnRow` to avoid surfacing PII in bulk responses.
