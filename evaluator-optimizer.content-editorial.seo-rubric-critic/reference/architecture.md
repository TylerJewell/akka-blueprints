# Architecture — seo-rubric-critic

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`AuditEndpoint` is the entry point. It writes an `ArticleSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `AuditWorkflow` per submission. The workflow alternates two agents — `RubricAgent` scores the article, `OptimizerAgent` revises it — with a deterministic guardrail check after every revision. Each step emits an event on `ArticleEntity`. `ArticlesView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ArticleSimulator` drips a sample article brief every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any scored round that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first draft fails the rubric and the second revision passes. The RubricAgent → guardrail → OptimizerAgent alternation is explicit. Each round's events are written to `ArticleEntity` before the next step begins so the UI's per-round timeline reconstructs the loop exactly.

## State machine

The article moves through two active states (`SCORING`, `REVISING`) and two terminal states (`APPROVED`, `FAILED_FINAL`). The self-loop on `REVISING` represents the guardrail-blocked path: an over-ceiling revision sends the workflow back to `reviseStep` with structured feedback. The `REVISING → SCORING` transition fires when a revision clears the guardrail and is ready to be re-scored. `SCORING → REVISING` fires when the rubric agent returns `FAIL` and there are rounds remaining. `FAILED_FINAL` is reached when the round ceiling is hit; both terminal states preserve every draft and every rubric score on the entity.

## Entity model

`ArticleEntity` is the system's source of truth; every transition writes one of seven event types. `SubmissionQueue` is the audit log of submissions. `ArticlesView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency and timeouts

- Per-step timeout: 60 s for `scoreStep` and `reviseStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxRounds` (default 4). A wall-clock guard could be added by lifting `maxRounds` into a duration; this is outside the default blueprint scope.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED_FINAL`, not in a hung workflow.
- Idempotency: `AuditEndpoint.submit` deduplicates on `(title, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(articleId, roundNumber)` so a tick that fires twice for the same round is a no-op.
- Per-round accounting: every round is appended to `Article.rounds` with its own draft, guardrail verdict, and (once produced) rubric score. The `roundNumber` is monotonic across the whole loop, including guardrail-blocked revisions that never reached the rubric agent.
