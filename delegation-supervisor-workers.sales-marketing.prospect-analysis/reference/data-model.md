# Data model

Authoritative record definitions. `/akka:implement` writes these exactly. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## `Prospect` (ProspectEntity state + ProspectsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Prospect UUID |
| `companyName` | `String` | no | Target company |
| `domain` | `Optional<String>` | yes | Primary domain, if supplied |
| `status` | `ProspectStatus` | no | Lifecycle state |
| `researchedAt` | `Optional<Instant>` | yes | When research completed |
| `companyProfile` | `Optional<CompanyProfile>` | yes | Research worker result |
| `analyzedAt` | `Optional<Instant>` | yes | When analysis completed |
| `orgStructureSummary` | `Optional<String>` | yes | Org summary from analysis |
| `decisionMakers` | `List<DecisionMaker>` | no (empty default) | Identified contacts, hints masked |
| `strategizedAt` | `Optional<Instant>` | yes | When strategy completed |
| `outreachStrategy` | `Optional<OutreachStrategy>` | yes | Strategy worker result |
| `completedAt` | `Optional<Instant>` | yes | When prospect reached COMPLETED |
| `stalledAt` | `Optional<Instant>` | yes | When monitor marked it STALLED |

`emptyState()` returns `Prospect.initial("", "")` — no `commandContext()` reference (Lesson 3).

## Worker result records

| Record | Fields |
|---|---|
| `CompanyProfile` | `String industry`, `String summary`, `String hqLocation`, `int employeeEstimate` |
| `OrgAnalysis` | `String orgStructureSummary`, `List<DecisionMaker> decisionMakers` |
| `DecisionMaker` | `String name`, `String title`, `String department`, `Optional<String> contactHint` |
| `OutreachStrategy` | `String approach`, `List<String> talkingPoints`, `String recommendedChannel` |

## `ProspectStatus` enum

`RESEARCHING`, `ANALYZING`, `STRATEGIZING`, `COMPLETED`, `STALLED`.

## Events (ProspectEntity)

| Event | Trigger |
|---|---|
| `ProspectQueued` | Workflow starts; initial state created |
| `CompanyResearched` | `recordResearch` — research worker returned a `CompanyProfile` |
| `OrgAnalyzed` | `recordAnalysis` — analysis worker returned an `OrgAnalysis`; sanitizer masks contact hints before this event persists |
| `StrategyDeveloped` | `recordStrategy` — strategy worker returned an `OutreachStrategy` |
| `ProspectStalled` | `markStalled` — monitor found the prospect stuck past threshold |

## RequestQueueEntity

| Event | Trigger |
|---|---|
| `AnalysisQueued` | `enqueueAnalysis(companyName, domain)` from endpoint or simulator |

## View row type

`ProspectsView` stores one `Prospect` row per prospect. Single query `getAllProspects` (`SELECT * AS prospects FROM prospects_view`) — no enum `WHERE` filter (Lesson 2); callers filter by status client-side.
