# Data model — Meeting Prep Briefer

Every record the generated system must define. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Briefing (BriefingEntity state + BriefingsView row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| id | String | no | Briefing UUID |
| meetingTopic | String | no | What the meeting is about |
| participants | List\<String> | no | Names submitted with the request |
| status | BriefingStatus | no | Lifecycle state (enum) |
| plannedAt | Optional\<Instant> | yes | When the research plan was recorded |
| researchPlan | Optional\<List\<String>> | yes | The planned subtasks (participant names + topic angles, flattened) |
| profiles | Optional\<List\<ParticipantProfile>> | yes | Redacted participant profiles |
| topicBrief | Optional\<TopicBrief> | yes | The researched topic brief |
| briefingDoc | Optional\<String> | yes | The composed markdown document |
| talkingPoints | Optional\<List\<String>> | yes | Suggested talking points |
| questions | Optional\<List\<String>> | yes | Suggested questions |
| readyAt | Optional\<Instant> | yes | When composition finished |
| failedAt | Optional\<Instant> | yes | When the briefing failed |
| failureReason | Optional\<String> | yes | Why it failed |

`emptyState()` returns `Briefing.initial("", "", List.of(), RECEIVED, …)` with all Optionals empty — no `commandContext()` reference (Lesson 3).

## Supporting records

| Record | Fields | Meaning |
|---|---|---|
| ParticipantProfile | `String name, String role, String redactedBackground` | One researched participant, PII-redacted |
| TopicBrief | `String summary, List<String> keyPoints` | Researched topic context |
| ResearchPlan | `List<String> participantNames, List<String> topicAngles` | Supervisor output |
| ComposedBriefing | `String briefingDoc, List<String> talkingPoints, List<String> questions` | Composer output |
| BriefingInputs | `String meetingTopic, List<ParticipantProfile> profiles, TopicBrief topicBrief` | Composer input |

## Status enum

```java
enum BriefingStatus { RECEIVED, PLANNING, RESEARCHING, COMPOSING, READY, FAILED }
```

## Events (BriefingEntity)

| Event | Trigger | Effect on state |
|---|---|---|
| BriefingRequested | requestBriefing command | Sets id, meetingTopic, participants; status RECEIVED → PLANNING |
| ResearchPlanned | recordPlan command | Sets researchPlan, plannedAt; status → RESEARCHING |
| ProfileAdded | addProfile command (post-sanitizer) | Appends a ParticipantProfile |
| TopicBriefAdded | setTopicBrief command | Sets topicBrief |
| BriefingComposed | recordComposition command | Sets briefingDoc, talkingPoints, questions, readyAt; status → READY |
| BriefingFailed | markFailed command | Sets failedAt, failureReason; status → FAILED |

## RequestQueue (EventSourcedEntity)

| Field / Event | Detail |
|---|---|
| Command | `enqueue(MeetingRequest)` |
| Event | `RequestQueued(String meetingTopic, List<String> participants)` |

## View row

`BriefingsView` row type is `Briefing` (above). One query: `getAllBriefings` → `SELECT * AS briefings FROM briefings_view`. No `WHERE status = :status` — Akka cannot auto-index the enum column (Lesson 2); status filtering happens client-side in the endpoint.
