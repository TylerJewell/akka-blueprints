# Data model — slack-lead-qualifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MemberJoinedEvent` | `eventId` | `String` | no | Unique event id from Slack or simulator. |
| | `userId` | `String` | no | Slack user id. |
| | `displayName` | `String` | no | Member's display name at join time. |
| | `email` | `String` | no | Raw email from Slack profile; *not sanitized*. |
| | `channelId` | `String` | no | Channel the member joined. |
| | `joinedAt` | `Instant` | no | Event timestamp. |
| `EnrichmentResult` | `company` | `String` | no | Inferred company; `"unknown"` sentinel if unavailable. |
| | `jobTitle` | `String` | no | Inferred job title; `"unknown"` sentinel if unavailable. |
| | `industry` | `String` | no | Short sector label; `"unknown"` sentinel if unavailable. |
| | `linkedInUrl` | `Optional<String>` | yes | Confirmed LinkedIn profile URL. |
| | `estimatedHeadcount` | `Optional<Integer>` | yes | Midpoint of inferred headcount range. |
| | `location` | `Optional<String>` | yes | City + country (e.g., `"Austin, US"`). |
| | `rawSearchSummary` | `String` | no | Verbatim excerpt, max 500 chars. |
| `SanitizedEnrichment` | `company` | `String` | no | Unchanged from enrichment. |
| | `jobTitle` | `String` | no | Unchanged from enrichment. |
| | `industry` | `String` | no | Unchanged from enrichment. |
| | `estimatedHeadcount` | `Optional<Integer>` | yes | Unchanged from enrichment. |
| | `piiCategoriesStripped` | `List<String>` | no | e.g., `["email", "phone", "location"]`. |
| `ScoringResult` | `score` | `int` | no | 0–100. |
| | `tier` | `LeadTier` | no | `HOT`, `WARM`, `COLD`, or `DISQUALIFIED`. |
| | `rationale` | `String` | no | Two to three sentence explanation. |
| | `matchedCriteria` | `List<String>` | no | ICP criteria met. |
| | `missedCriteria` | `List<String>` | no | ICP criteria not met or unavailable. |
| `SlackPostDraft` | `channel` | `String` | no | Target Slack channel id. |
| | `headline` | `String` | no | First line of the post. |
| | `body` | `String` | no | Post body text. |
| | `scoreBadge` | `String` | no | e.g., `"82 · HOT"`. |
| | `draftedAt` | `Instant` | no | When the draft was assembled. |
| `Lead` (entity state) | `leadId` | `String` | no | Derived from `eventId`. |
| | `event` | `MemberJoinedEvent` | no | Raw event captured at create. |
| | `enrichment` | `Optional<EnrichmentResult>` | yes | Populated after `LeadEnriched`. |
| | `sanitized` | `Optional<SanitizedEnrichment>` | yes | Populated after `LeadSanitized`. |
| | `scoring` | `Optional<ScoringResult>` | yes | Populated after `LeadScored`. |
| | `draft` | `Optional<SlackPostDraft>` | yes | Populated after `PostDrafted`. |
| | `slackTimestamp` | `Optional<String>` | yes | Slack message ts after post. |
| | `status` | `LeadStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MemberJoinedReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

## Enums

`LeadTier`: `HOT`, `WARM`, `COLD`, `DISQUALIFIED`.

`LeadStatus`: `RECEIVED`, `ENRICHED`, `SANITIZED`, `SCORED`, `POSTING`, `POSTED`, `SUPPRESSED`, `FAILED`.

## Events (`LeadEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MemberJoinedReceived` | `event` | → RECEIVED |
| `LeadEnriched` | `enrichment` | → ENRICHED |
| `LeadSanitized` | `sanitized` | → SANITIZED |
| `LeadScored` | `scoring` | → SCORED |
| `PostDrafted` | `draft` | → POSTING |
| `LeadPosted` | `slackTimestamp` | → POSTED (terminal) |
| `LeadSuppressed` | `reason: String` | → SUPPRESSED (terminal) |
| `LeadFailed` | `reason: String` | → FAILED (terminal) |

## Events (`SlackEventQueue`)

| Event | Payload |
|---|---|
| `MemberJoinedReceived` | `event` (the raw, pre-sanitization payload — audit log) |

## View row

`LeadRow` mirrors `Lead` minus the raw `event.email` field (the audit log retains that in `SlackEventQueue`). The UI fetches the full `Lead` on-demand via `GET /api/leads/{id}`.
