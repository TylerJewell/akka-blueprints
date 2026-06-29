# Architecture — graphrag-assistant

The four diagrams in [`../PLAN.md`](../PLAN.md) describe one request/response
agent over an in-memory graph index. This file narrates them.

## Component graph

`QueryEndpoint` is the front door. On `POST /api/ask` it creates a `QueryEntity`,
then calls `ResearchAgent.answer`. The agent owns two function tools —
`localSearch` and `globalSearch` — both backed by the `CorpusIndex`
KeyValueEntity. The agent picks one tool per question (entity-level vs.
community-level retrieval), then drafts an answer. `QueryEndpoint` applies the
PII sanitizer to retrieved chunks before the prompt and the grounding guardrail
to the drafted answer, then records the terminal state on `QueryEntity`.
`QueriesView` projects entity events into a read model the endpoint queries and
streams over SSE. `IndexBuilder` (TimedAction) builds the index from bundled docs
once at startup. `AppEndpoint` serves the single-file UI.

## Interaction sequence

The primary journey is one question end to end: receive → retrieve → sanitize →
draft → ground-check → record. There is no human pause — this is request/response,
not a workflow. The only branch is the grounding guardrail, which routes the
query to `ANSWERED` or `BLOCKED`.

## State machine

A `QueryEntity` has three states: `RECEIVED` on creation, then exactly one
terminal transition to `ANSWERED` (grounded) or `BLOCKED` (ungrounded). No
back-edges; each query is answered once.

## Entity model

`QueryEntity` emits `QueryReceived`, `AnswerRecorded`, and `AnswerBlocked`, which
`QueriesView` projects into one row per query. `CorpusIndex` holds the parsed
graph (entity-level chunks for local search, community summaries for global
search) built from the `sample-docs/*.md` set; it is a KeyValueEntity, not
event-sourced, because the index is rebuilt wholesale rather than evolved by
events.
