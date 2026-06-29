# User journeys — book-writer-fanout

Acceptance tests as numbered journeys. Each must pass against the generated system.

## J1 — Submit a topic and get an outline

- **Preconditions:** service running on `http://localhost:9537`; a model provider configured
  (or mock).
- **Steps:** open the App UI tab, type a book topic, click Submit.
- **Expected state:** `POST /api/books` returns a `bookId`; the book appears over SSE in
  `REQUESTED`, then within ~30 s transitions to `OUTLINED` with a non-empty `title` and a
  chapter list of 3–5 entries, each `PENDING`.
- **Done when:** the expanded book shows the title and numbered chapters.

## J2 — Chapters draft in sequence

- **Preconditions:** J1 reached `OUTLINED`.
- **Steps:** none — the workflow proceeds automatically.
- **Expected state:** the book moves to `WRITING`; each chapter transitions `DRAFTING` →
  `DRAFTED`; `chaptersDrafted` increments toward `chapterCount`; a `qualityScore` appears
  beside each drafted chapter.
- **Done when:** every chapter is `DRAFTED` (or `APPROVED`) with a non-empty draft.

## J3 — Consolidate into a manuscript

- **Preconditions:** every chapter drafted.
- **Steps:** none — consolidation runs automatically.
- **Expected state:** the book moves `CONSOLIDATING` → `COMPLETED`; the `manuscript` field is
  non-empty and contains every chapter title in order.
- **Done when:** the App UI renders the full manuscript markdown.

## J4 — Guard blocks an off-topic search

- **Preconditions:** a book is in `WRITING`.
- **Steps:** issue a `POST /api/search` with a query unrelated to the book topic, or one
  matching the injection denylist (e.g. `ignore previous instructions`).
- **Expected state:** the endpoint returns `403 { "refused": true, "reason": ... }`; no
  results are returned and nothing from the query reaches a chapter draft.
- **Done when:** the guard refuses and the chapter still drafts from the agent's own knowledge.

## J5 — Completeness gate blocks an incomplete book

- **Preconditions:** a book where one chapter has empty markdown (force by rejecting one
  chapter below the eval threshold and preventing a successful redraft).
- **Steps:** let the workflow reach `consolidateStep`.
- **Expected state:** the completeness gate fails; the book records `BookFailed` with a
  reason and reaches `FAILED` rather than `COMPLETED`; no partial manuscript is published as
  complete.
- **Done when:** the book is `FAILED` with a non-empty `failureReason`.

## J6 — Background load from the simulator

- **Preconditions:** service running, no UI interaction.
- **Steps:** wait one timer interval (~45 s).
- **Expected state:** `RequestSimulator` enqueues a topic from
  `sample-events/book-topics.jsonl`; a new book appears and runs the full flow.
- **Done when:** a book the user never submitted reaches `COMPLETED`.
