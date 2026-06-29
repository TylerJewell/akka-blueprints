# User journeys — editorial-desk

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: topic to published article

**Preconditions:** Service running on the declared port (9934); a model-provider key set, or the mock LLM selected during scaffolding. The writer roster (`writer-1`, `writer-2`) is running and idle-polling the section board.

**Steps:**
1. Open `http://localhost:9934/`. The App UI tab is visible.
2. In the form, enter the topic "How tides work". Click Submit.
3. A `storyId` is returned; the story appears with status `SUBMITTED`.

**Expected:**
- The editor-in-chief assigns a brief; the story moves to `ASSIGNED` and the brief angle appears on the card.
- The research desk runs: the lead plans subtopics, researcher instances each write a note into the workspace, and the lead synthesises a digest; the story moves through `RESEARCHING` to `RESEARCHED` and the digest summary appears.
- The writing desk runs: the lead plans sections, section cards appear in the section board's `Open` group, and the story moves to `WRITING`. Writers claim sections (each claimed card shows a writer id) and move them `Open` → `Claimed` → `Written`. The story-progress line tracks the count.
- When every section is `Written`, the draft assembles and the story moves to `DRAFTED`.
- The review panel scores the draft; with all axes passing the verdict is `PASS`, the story moves through `REVIEWING` to `APPROVED`, the editor-in-chief assembles the article, and the story moves to `PUBLISHED` with a generated URL.
- Three stage-eval score chips (research / draft / review) appear on the card.
- Both boards update live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two writers idle and at least one `OPEN` section on the board.

**Steps:**
1. Submit a story so the writing desk plans several sections that land on the board at once.

**Expected:**
- Each section shows exactly one writer id; no section is ever shown claimed by two writers.
- Writers that lose a claim race return to polling and pick up different sections (or idle if none remain).
- The claim count per section is one.

## J3 — Bounded revision loop

**Preconditions:** As J1. Under the mock LLM, the `reviewer.json` set includes at least one `REVISE` note naming a section title; with a real model, a draft with a real flaw on one axis.

**Steps:**
1. Submit a story whose draft draws a `REVISE` from one of the review axes.
2. Watch the story.

**Expected:**
- The review verdict is `REVISE`; the named section(s) reset to `OPEN` on the board and the story returns to `WRITING` with `revisionCount` now 1.
- A writer claims and rewrites the reset section; the draft reassembles and the panel scores it again.
- On the second pass the story proceeds to publish (a second `REVISE` is accepted rather than looping again), so the pipeline terminates.

## J4 — Refused document-workspace write

**Preconditions:** As J1. A stage whose agent attempts a write outside its assignment (under the mock LLM, a `researcher.json` entry whose note targets a mismatched story id; with a real model, a prompt-injected topic).

**Steps:**
1. Submit the story; watch the offending stage.

**Expected:**
- The before-tool-call guardrail refuses the `DocumentTools` write; it never lands.
- The stage is recorded with the guardrail reason; the agent returns without a partial write. For a section write, the `WriterWorkflow` releases the claim and the section returns to `OPEN` for another writer.
- No content is written into a story or section the agent was not assigned, and no write lands on an already-published story.

## J5 — Post-publication compliance review

**Preconditions:** A story has reached `PUBLISHED` (run J1 first).

**Steps:**
1. On the published story's card, open the compliance-review box.
2. Enter a reviewer id, pick `FLAGGED`, add a comment, and Submit (or `POST /api/stories/{id}/compliance-review`).

**Expected:**
- The review is recorded against the story; the card shows the recorded `FLAGGED` outcome and comment.
- The story's status stays `PUBLISHED` — the review does not change it.
- Posting a compliance review to a story that is not yet `PUBLISHED` returns `409` and records nothing.
