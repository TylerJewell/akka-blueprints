# Architecture — book-writer-fanout

The system turns one book topic into a finished manuscript by composing a planning agent, a
per-chapter writing pass, and a consolidation step. The four diagrams below are the mermaid
source rendered on the Architecture tab. Each must carry the Lesson-24 theme variables plus
the CSS overrides for state-box labels and edge-label overflow, or the state names render
black-on-black and transition labels clip.

## Component graph

The flow starts at either `BookEndpoint` (a user POST) or `RequestSimulator` (a timed drip),
both of which enqueue a topic on `BookRequestQueue`. `BookRequestConsumer` reacts to the
queued event and starts one `BookWritingWorkflow` per topic. The workflow calls
`OutlineAgent` once, then `ChapterAgent` once per planned chapter; `ChapterAgent` reaches the
guarded `WebSearchEndpoint` for on-topic lookups. Book and chapter state live in
`BookEntity` and `ChapterEntity`; their events project into `BooksView`, which `BookEndpoint`
lists and streams. `ChapterEvalConsumer` scores each chapter out of band. See the
`flowchart TB` in `PLAN.md`.

## Interaction sequence

The primary journey: submit → outline → per-chapter draft (with guarded search and an
out-of-band eval) → completeness gate → consolidate → complete. The loop over chapters is the
composite-multi-team fan-out — each iteration is a fresh writing pass with its own agent
session and its own `ChapterEntity`. See the `sequenceDiagram` in `PLAN.md`.

## State machine

`BookEntity` moves `REQUESTED → OUTLINED → WRITING → CONSOLIDATING → COMPLETED`, with a
`FAILED` terminal reached from `CONSOLIDATING` (completeness gate fails) or `WRITING` (step
retries exhausted). See the `stateDiagram-v2` in `PLAN.md`.

## Entity model

One `BOOK` contains many `CHAPTER` rows; both project into the single `BOOKS_VIEW` row (the
book row embeds a chapter summary array). `BOOK_REQUEST_QUEUE` records inbound topics. See the
`erDiagram` in `PLAN.md`.

## Why these primitives

- The fan-out is a `Workflow` loop rather than parallel sub-workflows so the canned 3–5
  chapter outlines stay inside one step budget; the directives note the split-by-index option
  for larger books.
- Per-chapter state is its own `EventSourcedEntity` so the eval can score a chapter
  independently of book progress and a retry can re-target a deterministic chapter id.
- The web-search facade is an in-process `HttpEndpoint` so the before-tool-call guard is a
  real Akka request handler, not a mock — the sample runs out of the box with no external
  search service.
