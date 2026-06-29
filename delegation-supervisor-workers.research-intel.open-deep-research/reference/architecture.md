# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A question enters through `ResearchEndpoint` (`POST /api/research`) or is dripped by `QuestionSimulator` every 60 seconds. Either path writes a `QuestionSubmitted` event onto `QuestionQueue`. `QuestionConsumer` subscribes to those events and starts one `ResearchWorkflow` per submission, keyed by `runId`.

The workflow is the supervisor. It asks `ResearchManager` to decompose the question into a `RetrievalPlan`, fans retrieval out to `WebBrowsingAgent` and `FileReaderAgent` in parallel, sanitizes PII from their outputs, then asks `ResearchManager` to synthesise. Every transition is written as a command to `ResearchRunEntity`, whose events project into `ResearchView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a run to score.

The `WebBrowsingAgent` has a `before-tool-call` guardrail that runs before each URL fetch. Disallowed URLs are blocked at the guardrail boundary and recorded on the run entity; allowed URLs proceed normally. This guardrail is not a workflow step — it is inline within the agent's tool dispatch loop.

## Interaction sequence

The sequence diagram traces the primary journey: plan, parallel retrieval fan-out, PII sanitization, synthesis, monitoring check, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The monitoring step is non-blocking: it emits an `OversightFlagged` event if the retry budget is exceeded but does not alter the final `ANSWERED` status.

## State machine

`ResearchRun` moves `QUEUED → IN_PROGRESS`, then to one of three terminal groups: `ANSWERED` (synthesis complete), `DEGRADED` (a worker timed out and synthesis ran on partial input), or `BLOCKED` (guardrail blocked all URLs and no content came back). `ANSWERED` accepts two further self-transitions: `RunEvalScored` when `EvalSampler` records a score, and `OversightFlagged` when the retry budget is exceeded.

## Entity model

`QuestionQueue` seeds one `ResearchRun` per question. A run owns at most one `WebBundle`, one `DocumentBundle`, and one `SynthesisedAnswer`. The view row mirrors the run with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). `oversightFlagged` is a primitive boolean, not Optional, because its absence is meaningful (false = no flag raised).
