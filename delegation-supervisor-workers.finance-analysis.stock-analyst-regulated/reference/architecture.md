# Architecture — Stock Analysis Team

Narrative around the four diagrams in `PLAN.md`. The generated UI renders the same mermaid sources on the Architecture tab.

## Component graph

The supervisor is `AnalysisWorkflow`. It receives a ticker either from `StockAnalysisEndpoint` (a user submission) or from `RequestConsumer` (a simulated request drained from `InboundRequestQueue`). The workflow delegates to four worker agents — `NewsResearchAgent`, `FilingsResearchAgent`, `SummaryAgent`, `RecommendationAgent` — and writes each result onto `AnalysisEntity`. `AnalysisView` projects the entity's events into a read model the endpoint queries and streams over SSE. `RequestSimulator` drips canned tickers; `DriftMonitor` samples the view and flags skewed recommendation distributions.

## Interaction sequence

The primary journey runs ticker in, recommendation out. Research fans out to the news and filings agents, then converges on the summary agent. The sanitize step scrubs fair-disclosure-sensitive phrasing before the recommendation agent runs. The before-agent-response guardrail on the recommendation agent enforces a disclaimer and a safety check. The recommendation is issued in `ISSUED_PENDING_REVIEW`, where a compliance reviewer can clear or retract it live.

## State machine

One analysis advances `QUEUED → RESEARCHING → SUMMARIZING → SANITIZING → RECOMMENDING → ISSUED_PENDING_REVIEW`, then terminates at `CLEARED` or `RETRACTED` by reviewer action. Any agent-calling step can fail to `FAILED` after retries.

## Entity model

`AnalysisEntity` is event-sourced; its events project to `AnalysisView`. `InboundRequestQueue` is a separate entity whose events drive `RequestConsumer`. The view row is the `Analysis` record with every nullable lifecycle field declared `Optional<T>`.
