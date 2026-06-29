# User journeys — Landing Page Generator

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit a concept and reach READY

**Preconditions:** service running on `http://localhost:9297/`; a model provider configured (or mock).
**Steps:**
1. In the App UI tab, enter "a budgeting app for freelancers" and click **Generate page** (or `POST /api/concepts`).
2. Observe the response `pageId`.
**Expected:** a page appears in `DRAFTING_COPY`, then advances `STRUCTURING → WRITING_CTA → REVIEWING → READY` over the SSE stream. The final `READY` page has a non-empty `headline`, `subhead`, four-to-six `sections`, both CTA blocks, and a `reviewScore`.

## J2 — A page fails the brand-safety gate

**Preconditions:** service running; the canned flagging concept present in `concepts.jsonl` (or a mock that returns a failing `ReviewResult`).
**Steps:**
1. Submit (or let the simulator drip) the concept that trips the brand-safety rubric.
**Expected:** the page advances to `REVIEWING`, then lands in `FLAGGED` with a non-empty `flagReason`. It is never marked `READY`. The App UI shows the flag reason in red.

## J3 — Background simulator seeds a concept

**Preconditions:** service freshly started; no UI interaction.
**Steps:**
1. Wait up to ~30s.
**Expected:** `ConceptSimulator` enqueues a concept from `concepts.jsonl`; `ConceptConsumer` starts a workflow; a new page appears in the list without any user action.

## J4 — All five tabs render

**Preconditions:** service running.
**Steps:**
1. Open each tab: Overview, Architecture, Risk Survey, Eval Matrix, App UI.
**Expected:** every tab shows content (no blank panel — Lesson 26). The Architecture tab renders four mermaid diagrams with readable state labels. The Eval Matrix tab shows control G1 with a red guardrail pill. The Risk Survey tab mutes `TO_BE_COMPLETED_BY_DEPLOYER` fields.

## J5 — Single-page fetch

**Preconditions:** at least one page exists.
**Steps:**
1. `GET /api/pages/{pageId}` for a known id.
**Expected:** the full `LandingPage` JSON, with `Optional` fields serialized as raw values or `null`.
