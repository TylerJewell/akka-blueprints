# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

An image batch enters through `AnalysisEndpoint` (`POST /api/analysis`) or is dripped by `BatchSimulator` every 90 seconds. Either path writes a `BatchSubmitted` event onto `BatchQueue`. `BatchRequestConsumer` subscribes to those events and starts one `AnalysisWorkflow` per submission, keyed by `jobId`.

The workflow is the supervisor. It first calls `PiiSanitizer` over each image sequentially, then calls `OpusCoordinator` to decompose the question into per-image instructions. It fans out to one `HaikuImageAnalyst` per image in parallel, collects the `ImageReport` objects, and calls `OpusCoordinator` again to synthesise. Every transition is written as a command to `AnalysisJobEntity`, whose events project into `AnalysisView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a job to score.

The two agent types serve distinct roles: `OpusCoordinator` handles open-ended reasoning that requires the larger model; `HaikuImageAnalyst` handles narrow, per-image description tasks where the smaller, faster model is sufficient. This split is the core cost/latency optimisation the blueprint demonstrates.

## Interaction sequence

The sequence diagram traces the primary journey: sanitize, decompose, parallel fan-out, join, synthesise, guardrail, persist. The `par` block is the delegation-supervisor-workers core — all image analysts run concurrently and the workflow joins their results. The sanitize step runs before the parallel fan-out to ensure every sub-agent receives a clean image. The guardrail decides between the `SYNTHESISED` and `BLOCKED` terminals.

## State machine

`AnalysisJob` moves `QUEUED → ANALYSING`, then to one of three terminals: `SYNTHESISED` (guardrail passed), `PARTIAL` (one or more sub-agents timed out), or `BLOCKED` (guardrail failed). `SYNTHESISED` accepts one further `JobEvalScored` self-transition when `EvalSampler` records a score.

The distinction between `PARTIAL` and `BLOCKED` matters: `PARTIAL` means the system ran correctly but could not collect all sub-agent outputs; `BLOCKED` means all outputs arrived but the synthesised answer failed the quality gate. A deployer should handle these two terminals differently — `PARTIAL` may warrant re-submission; `BLOCKED` warrants investigation of the synthesised content.

## Entity model

`BatchQueue` seeds one `AnalysisJob` per submission. A job owns a list of `SanitizedImage` objects (one per input image), a list of `ImageReport` objects (one per successful sub-agent call), and at most one `SynthesisedAnswer`. The view row mirrors the job with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6).
