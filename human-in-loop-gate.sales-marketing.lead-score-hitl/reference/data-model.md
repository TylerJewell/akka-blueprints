# Data Model — Lead Score HITL

Every nullable lifecycle field is `Optional<T>` (Lesson 6); the View materializer rejects non-optional `null` fields. Akka serializes `Optional<T>` as the raw value or `null`.

## `Lead` (entity state and View row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Lead id (UUID) |
| `company` | `Optional<String>` | yes | Company name |
| `contactName` | `Optional<String>` | yes | Sanitized contact name (PII redacted) |
| `rawSource` | `Optional<String>` | yes | Original submission text |
| `status` | `LeadStatus` | no | Lifecycle status |
| `collectedAt` | `Optional<Instant>` | yes | When collection completed |
| `enrichedProfile` | `Optional<String>` | yes | Profile from LeadCollectionAgent |
| `analyzedAt` | `Optional<Instant>` | yes | When analysis completed |
| `analysisSummary` | `Optional<String>` | yes | Summary from LeadAnalysisAgent |
| `scoredAt` | `Optional<Instant>` | yes | When scoring completed |
| `score` | `Optional<Integer>` | yes | 0–100 priority score |
| `scoreRationale` | `Optional<String>` | yes | One-line score rationale |
| `shortlisted` | `Optional<Boolean>` | yes | True once ranked into the top tier |
| `reviewedAt` | `Optional<Instant>` | yes | When a human decided |
| `reviewedBy` | `Optional<String>` | yes | Reviewer id |
| `reviewDecision` | `Optional<String>` | yes | `APPROVED` or `REJECTED` |
| `reviewComment` | `Optional<String>` | yes | Reviewer note / reject reason |
| `outreachDraftedAt` | `Optional<Instant>` | yes | When outreach was drafted |
| `outreachSubject` | `Optional<String>` | yes | Drafted email subject |
| `outreachBody` | `Optional<String>` | yes | Drafted email body |

`initial(String id, String company)` returns a `Lead` with `status = NEW` and every `Optional` empty.

## `LeadStatus` enum

`NEW · COLLECTED · ANALYZED · SCORED · SHORTLISTED · APPROVED · REJECTED · CONTACTED`

## `LeadEvent` (sealed interface)

| Event | Trigger | Sets |
|---|---|---|
| `LeadCollected(leadId, company, contactName, enrichedProfile, timestamp)` | `collectStep` after sanitization | `COLLECTED`, profile, collectedAt |
| `LeadAnalyzed(leadId, summary, timestamp)` | `analyzeStep` | `ANALYZED`, analysisSummary, analyzedAt |
| `LeadScored(leadId, score, rationale, timestamp)` | `scoreStep` | `SCORED`, score, scoreRationale, scoredAt |
| `LeadShortlisted(leadId, timestamp)` | `LeadRankingMonitor` | `SHORTLISTED`, shortlisted=true |
| `LeadApproved(leadId, reviewedBy, comment, timestamp)` | approve command | `APPROVED`, review fields |
| `LeadRejected(leadId, reviewedBy, reason, timestamp)` | reject command | `REJECTED`, review fields |
| `OutreachDrafted(leadId, subject, body, timestamp)` | `outreachStep` | `CONTACTED`, outreach fields |

`InboundLeadQueue` emits `InboundLeadQueued(leadId, rawSource, company, contactName, timestamp)`.

## Auxiliary records

```java
record CollectedLead(String company, String contactName, String enrichedProfile) {}
record LeadAnalysis(String summary) {}
record LeadScore(int score, String rationale) {}
record OutreachEmail(String subject, String body) {}
record ReviewDecision(String reviewedBy, String decision, String comment) {}
```

## View row type

`LeadsView` uses `Lead` as its row type. One query `getAllLeads` (`SELECT * AS leads FROM leads_view`) with no `WHERE status` clause — status filtering and top-3 ranking happen client-side (Lesson 2). `streamAllLeads` backs the SSE endpoint.
