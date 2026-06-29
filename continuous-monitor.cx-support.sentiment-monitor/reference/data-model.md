# Data model — sentiment-monitor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IssueComment` | `commentId` | `String` | no | Unique id from the Linear feed source. |
| | `issueId` | `String` | no | Linear issue identifier (e.g., `ENG-4821`). |
| | `authorId` | `String` | no | Author identifier; not displayed in alerts. |
| | `body` | `String` | no | Raw comment body. |
| | `postedAt` | `Instant` | no | When the comment was posted. |
| `SentimentScore` | `score` | `int` | no | Integer −5 to +5. |
| | `confidence` | `String` | no | `"high" \| "medium" \| "low"`. |
| | `reason` | `String` | no | One short sentence. |
| `ScoredComment` | `commentId` | `String` | no | — |
| | `comment` | `IssueComment` | no | The original comment. |
| | `score` | `SentimentScore` | no | The scorer's output. |
| | `scoredAt` | `Instant` | no | When scoring completed. |
| `TrendWindow` | `issueId` | `String` | no | — |
| | `issueTitle` | `String` | no | — |
| | `recentScores` | `List<SentimentScore>` | no | Last N scores (window size configurable). |
| | `runningAverage` | `double` | no | Rolling average of all scores on the thread. |
| | `consecutiveNegativeCount` | `int` | no | Current run of negative-score comments. |
| `AlertSummary` | `headline` | `String` | no | ≤ 120 chars; names the issue and signal. |
| | `detail` | `String` | no | Two sentences: average and consecutive count. |
| | `recentReasons` | `List<String>` | no | Up to 3 verbatim/paraphrased score reasons. |
| | `generatedAt` | `Instant` | no | When the agent finished. |
| `AlertDecision` | `dispatched` | `boolean` | no | true = alert sent, false = blocked. |
| | `channelId` | `String` | no | Target channel ID. |
| | `silencedBy` | `Optional<String>` | yes | Set when a human silences the thread. |
| | `decidedAt` | `Instant` | no | When dispatch or rejection was recorded. |
| `LabeledScore` | `commentId` | `String` | no | — |
| | `predictedScore` | `int` | no | The agent's score. |
| | `groundTruthScore` | `int` | no | Human-labeled ground truth. |
| `DriftEvalResult` | `meanAbsoluteError` | `double` | no | Mean of \|predicted − truth\|. |
| | `sampleSize` | `int` | no | Number of entries evaluated. |
| | `verdict` | `String` | no | `"NO_DRIFT" \| "MINOR_DRIFT" \| "SIGNIFICANT_DRIFT"`. |
| | `evaluatedAt` | `Instant` | no | When the eval run completed. |
| `IssueThread` (entity state) | `issueId` | `String` | no | — |
| | `issueTitle` | `String` | no | — |
| | `scoredComments` | `List<ScoredComment>` | no | Full history. |
| | `runningAverage` | `double` | no | — |
| | `trendLabel` | `TrendLabel` | no | See enum. |
| | `consecutiveNegativeCount` | `int` | no | — |
| | `lastAlert` | `Optional<AlertDecision>` | yes | Most recent dispatch decision. |
| | `status` | `ThreadStatus` | no | See enum. |
| | `firstSeenAt` | `Instant` | no | When `ThreadOpened` was emitted. |
| | `criticalAt` | `Optional<Instant>` | yes | When thread first reached CRITICAL. |

## Enums

`TrendLabel`: `STABLE`, `DECLINING`, `CRITICAL`.
`ThreadStatus`: `ACTIVE`, `SILENCED`, `RESOLVED`.

## Events (`IssueThreadEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ThreadOpened` | `issueId, issueTitle` | → ACTIVE |
| `CommentScored` | `scoredComment` | (recalculates average + consecutive count) |
| `TrendUpdated` | `trendLabel, consecutiveNegativeCount, runningAverage` | Updates trend fields |
| `CriticalTrendDetected` | `consecutiveNegativeCount, runningAverage` | Sets `criticalAt`; signals AlertDispatchWorkflow |
| `AlertDispatched` | `alertDecision` | Updates `lastAlert` |
| `AlertSilenced` | `silencedBy, decidedAt` | → SILENCED; updates `lastAlert.silencedBy` |
| `ThreadResolved` | `resolvedBy, resolvedAt` | → RESOLVED (terminal) |

## Events (`CommentQueue`)

| Event | Payload |
|---|---|
| `CommentReceived` | `comment` (raw `IssueComment` — used by the audit log) |

## Events (`SentimentEvalEntity`)

| Event | Payload |
|---|---|
| `DriftEvalRecorded` | `result: DriftEvalResult` |

## View row

`IssueThreadRow` mirrors `IssueThread` with `scoredComments` omitted from the list query (full list returned only by `GET /api/sentiment/threads/{issueId}`). Includes `latestScore: Optional<SentimentScore>` derived from the last entry in `scoredComments`. Also includes `driftVerdict: Optional<String>` joined from `SentimentEvalEntity`'s most recent `DriftEvalRecorded`.
