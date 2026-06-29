# Data model — morning-email-debrief

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ReceivedEmail` | `emailId` | `String` | no | Unique id, generated per email by the mailbox source. |
| | `runId` | `String` | no | Groups all emails from one polling batch. |
| | `from` | `String` | no | Raw sender address; *not yet sanitized*. |
| | `subject` | `String` | no | Raw subject line. |
| | `body` | `String` | no | Raw email body. |
| | `receivedAt` | `Instant` | no | When the poller delivered it. |
| `SanitizedEmailPayload` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g., `["email","name","phone"]`. |
| `DebriefEntry` | `emailId` | `String` | no | Back-reference to the source email. |
| | `oneLinerSummary` | `String` | no | ≤ 120 chars. |
| | `priority` | `Priority` | no | One of `HIGH`, `MEDIUM`, `LOW`. |
| | `category` | `String` | no | Freeform label e.g. `"legal"`, `"ops"`. |
| `MorningDebrief` | `runId` | `String` | no | — |
| | `narrativeSummary` | `String` | no | 3–5 sentences. |
| | `entries` | `List<DebriefEntry>` | no | All per-email entries for the run. |
| | `totalEmailCount` | `int` | no | Count of emails in the batch. |
| | `assembledAt` | `Instant` | no | When the assembler finished. |
| `EvalResult` | `score` | `Integer` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| `EmailRecord` (entity state) | `emailId` | `String` | no | — |
| | `runId` | `String` | no | — |
| | `incoming` | `ReceivedEmail` | no | Captured once at create; audit-only. |
| | `sanitized` | `Optional<SanitizedEmailPayload>` | yes | Populated after EmailSanitized. |
| | `entry` | `Optional<DebriefEntry>` | yes | Populated after EmailSummarised. |
| | `status` | `EmailStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When EmailReceived emitted. |
| `DebriefRun` (entity state) | `runId` | `String` | no | — |
| | `totalEmails` | `int` | no | How many emails were submitted. |
| | `summarisedCount` | `int` | no | How many have reached SUMMARISED. |
| | `debrief` | `Optional<MorningDebrief>` | yes | Populated after DebriefReady. |
| | `evalScore` | `Optional<Integer>` | yes | Populated after DebriefEvalScored. |
| | `evalRationale` | `Optional<String>` | yes | Populated after DebriefEvalScored. |
| | `status` | `DebriefStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When DebriefCreated emitted. |
| | `readyAt` | `Optional<Instant>` | yes | Terminal success timestamp. |

## Enums

`Priority`: `HIGH`, `MEDIUM`, `LOW`.

`EmailStatus`: `RECEIVED`, `SANITIZED`, `SUMMARISED`.

`DebriefStatus`: `CREATED`, `ASSEMBLING`, `READY`, `FAILED`.

## Events (`EmailEntity`)

| Event | Payload | Transition |
|---|---|---|
| `EmailReceived` | `incoming` | → RECEIVED |
| `EmailSanitized` | `sanitized` | → SANITIZED |
| `EmailSummarised` | `entry` | → SUMMARISED (terminal for email) |

## Events (`DebriefEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DebriefCreated` | `runId, totalEmails` | → CREATED |
| `DebriefAssembling` | — | → ASSEMBLING |
| `DebriefReady` | `debrief: MorningDebrief` | → READY (terminal success) |
| `DebriefFailed` | `reason: String` | → FAILED (terminal error) |
| `DebriefEvalScored` | `score, rationale` | no status change; populates `evalScore` + `evalRationale` |

## Events (`EmailQueue`)

| Event | Payload |
|---|---|
| `EmailReceived` | `incoming` (raw, pre-sanitization payload — audit log only) |

## View rows

`EmailRow` mirrors `EmailRecord` minus the raw `incoming` payload (audit log retains that). The UI fetches it on-demand via `GET /api/email/{emailId}`.

`DebriefRow` mirrors `DebriefRun` minus the full `MorningDebrief` body; the full debrief is fetched via `GET /api/debrief/{runId}` when the user opens the detail panel.
