# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A research question enters through `ReportEndpoint` (`POST /api/reports`) or is dripped by `QuestionSimulator` every 60 seconds. Either path writes a `QuestionSubmitted` event onto `QuestionQueue`. `QuestionConsumer` subscribes to those events and starts one `DeepResearchWorkflow` per submission, keyed by `reportId`.

The workflow is the supervisor. It asks `ResearchSupervisor` to decompose the question into subqueries, then fans out: for each subquery it runs `SearchWorker` → `ExtractionWorker` → `SummaryWorker` in a three-step pipeline. The subquery pipelines run in parallel across all subqueries. Once all pipelines complete (or time out), the workflow asks `ResearchSupervisor` to synthesise. Every transition is written as a command to `ResearchReportEntity`, whose events project into `ReportView`. The endpoint reads and streams the view; `CitationEvalSampler` reads it to pick a report to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel subquery fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — each subquery pipeline runs concurrently. The guardrail step decides between the `SYNTHESISED` and `BLOCKED` terminals.

## State machine

`ResearchReport` moves `PLANNING → IN_PROGRESS`, then to one of three terminals: `SYNTHESISED` (guardrail passed), `DEGRADED` (one or more subquery search steps timed out), or `BLOCKED` (guardrail failed). `SYNTHESISED` accepts one further `ReportEvalScored` self-transition when `CitationEvalSampler` records a citation-grounding score.

## Entity model

`QuestionQueue` seeds one `ResearchReport` per question. A report owns zero or more `SubquerySummary` records (one per subquery that completed), at most one `SynthesisedReport`, and a flat `citationList` derived from the subquery supporting claims. The view row mirrors the report with `Optional<T>` on every field that is null before its transition (Lesson 6).
