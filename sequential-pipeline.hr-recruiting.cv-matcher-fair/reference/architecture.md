# Architecture — Fair CV Matcher

The system is a sequential pipeline. A submission enters through an endpoint or the simulator, lands on an intake queue, and a consumer starts one workflow per candidate. The workflow runs four steps in fixed order, each writing an event onto the candidate's event-sourced record. A view projects those events for the UI, and a periodic monitor watches fairness across the matched population. The four diagrams in `PLAN.md` are the source of truth; this file narrates them.

## Component graph

Submissions reach `CvIntakeQueue` two ways: a user `POST` through `MatchingEndpoint`, or `CvSimulator` dripping a sample CV every 30 seconds. `CvIntakeConsumer` subscribes to the queue's events and starts a `MatchingWorkflow` per submission. The workflow drives `ExtractionAgent` and `MatchingAgent` and writes every lifecycle change to `CandidateEntity`. `MatchesView` projects the entity for the endpoint to list and stream. `FairnessMonitor` reads the view and writes drift flags back. `AppEndpoint` serves the single-file UI.

## Interaction sequence

The primary journey is a submission running to `MATCHED`, then a reviewer recording oversight. The sequence shows the agent calls bracketing the deterministic sanitize step, and the `Note` blocks mark where special-category signals are stripped and where the candidate is released for human-on-the-loop review.

## State machine

A candidate moves `SUBMITTED → EXTRACTED → SANITIZED → MATCHED → REVIEWED`. Each transition is an event applied by `CandidateEntity`. There is no rejection or escalation branch in this pipeline; the review step is non-blocking, so a candidate may sit in `MATCHED` until a reviewer acts.

## Entity model

`Candidate` is the aggregate. It owns one `CvProfile`, a list of `RedactedSignal` entries produced by the sanitize step, and a list of `MatchResult` entries produced by the matching step. The `slice` label on the candidate is what `FairnessMonitor` groups by.
