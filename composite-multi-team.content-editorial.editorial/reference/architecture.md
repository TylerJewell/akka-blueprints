# Architecture — editorial-desk

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EditorialEndpoint` is the entry point. A submitted topic is logged as a `StorySubmitted` event on `StoryQueue` (event-sourced for audit). `StoryRequestConsumer` subscribes to that queue, creates a `DocumentEntity`, and starts an `EditorialWorkflow` keyed by the story id. That workflow is the editor-in-chief: it runs the story through five stages, writing each stage's result onto the shared `DocumentEntity` before the next begins.

The three desks under the pipeline each run a different internal coordination capability over the same shared workspace:

- **Research desk — delegation.** `researchStep` runs `ResearchLead` to plan subtopics, fans out one `Researcher` instance per subtopic (each writing a note into the workspace through `DocumentTools`), then runs `ResearchLead` again to synthesise the notes into a digest.
- **Writing desk — a team over a shared list.** `writingStep` runs `WritingLead` to plan sections, writes one `SectionEntity` per section onto the board, and then waits. The writer roster (`writer-1`, `writer-2`) runs independently: `Bootstrap` starts one `WriterWorkflow` per writer, each polling `SectionBoardView`, claiming an open section atomically, running the `Writer` agent, and marking the section written.
- **Review desk — moderation.** `reviewStep` runs one `Reviewer` instance per axis (`factcheck`, `style`, `legal`), each scoring the draft and writing a note; the deterministic `ModerationRule` turns the panel's notes into a pass-or-revise verdict.

`StageEvalConsumer` subscribes to the document's stage events and records a non-blocking quality eval after each stage result. Two TimedActions sit alongside: `StorySimulator` drips sample topics; `StuckSectionMonitor` releases stranded section claims. `AppEndpoint` serves the embedded UI and the metadata the tabs read.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → assign → research (delegate and synthesise) → plan sections → writers claim and fill the board → assemble the draft → the panel scores → a passing verdict assembles and publishes the article. Two `Note over` blocks mark the asynchronous handoffs: the writer loops are already polling the board before any section exists, and the writing stage waits by polling rather than blocking. A third marks where the before-agent-response guardrail vets the final article.

## State machine

`DocumentEntity` is the spine of the pipeline. It moves `SUBMITTED → ASSIGNED → RESEARCHING → RESEARCHED → WRITING → DRAFTED → REVIEWING`, and then either `APPROVED → PUBLISHED` on a passing review or back to `WRITING` for one bounded revision round on a `REVISE` verdict. Two self-transitions matter: `recordPublishBlock` keeps the document `APPROVED` when the output guardrail refuses the article (nothing is published), and `recordComplianceReview` keeps the document `PUBLISHED` when a post-publication compliance review lands — the reviewer is on the loop, not gating it.

## Entity model

`DocumentEntity` is the shared workspace and the source of truth for a story; every stage writes one of ten event types, and `DocumentBoardView` is its read-side projection. `SectionEntity` is the writing-team's coordination point — one per section, with an atomic `claim` that the single-writer guarantee resolves to exactly one winner under contention; `SectionBoardView` is the shared list the writers poll. `StoryQueue` is the audit log of submissions. `StageEvalConsumer` reads document events to score stages; it writes its eval back through a command, not by mutating state directly.

## Concurrency & timeouts

- The top-level pipeline is sequential delegation; the writing stage hands off to an independent team and waits on the board.
- Atomic claim through `SectionEntity`'s single writer is the writing-team primitive — no lock, no external queue.
- Per-step timeouts on the agent-calling steps: `briefStep` 60 s, `researchStep` 120 s (it fans out several researcher calls), `reviewStep` 90 s, `publishStep` 60 s, and `WriterWorkflow.writeStep` 90 s.
- The writing wait and idle writers are paused workflows with a 5 s resume timer, not busy loops.
- The revision loop runs at most once, so the pipeline always terminates.
- `StuckSectionMonitor` releases a section claimed-but-idle for more than two minutes so a failed writer does not strand the board.
- `ModerationRule` and `StageEvaluator` are deterministic pure functions (no LLM call), so the review verdict and the stage scores are reproducible.
