# User journeys — shared-memory-multi-agent

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path: block seeded to consolidated

**Preconditions:** Service running on declared port (9748); a model-provider key set, or the mock LLM selected during scaffolding. The writer roster (`writer-1`, `writer-2`, `writer-3`) is running and cycling.

**Steps:**
1. Open `http://localhost:9748/`. The App UI tab is visible.
2. In the form, enter block name "project-glossary" and a one-line initial content describing the project. Click Seed.
3. A `blockId` is returned; the block appears with status `SEEDED` then quickly `ACTIVE`.

**Expected:**
- Within a few cycles each writer contributes a fragment; the block's pending fragment count increments with each write.
- Fragment cards appear in the fragment panel with status `PENDING` (or `SANITIZED` if the sanitizer fired).
- When the pending count reaches the consolidation threshold (or the scheduler fires), the block moves to `CONSOLIDATING` — new write calls are refused by the guardrail during this window.
- A `ConsolidatedSnapshot` is committed; the block moves to `CONSOLIDATED`. The board card shows the consolidation summary.
- All merged fragments update to `MERGED` status in the fragment panel.
- The board and fragment panel update live via SSE throughout, with no page reload.

## J2 — Concurrent writes: no fragment duplication or loss

**Preconditions:** As J1, with at least two writers idle and one `ACTIVE` block.

**Steps:**
1. Observe two or more writers contributing to the same block within the same 30 s window.

**Expected:**
- Each fragment is recorded exactly once in `FragmentEntity`; no `fragmentId` appears more than once in the fragment panel.
- Each fragment carries a unique author id.
- The block's `pendingFragmentCount` increments by exactly one per fragment — no double-count.
- After consolidation, `mergedFragmentIds` in the snapshot lists each fragment id once.

## J3 — Guardrail: write to protected section refused

**Preconditions:** As J1. Under the mock LLM, the `writer-agent.json` set includes an entry whose content references `_system`; with a real model, a block seed whose description includes an instruction to modify the `_system` section.

**Steps:**
1. Seed a block that causes a writer to attempt a write to a protected section.
2. Watch the offending fragment.

**Expected:**
- The before-tool-call guardrail (G1) refuses the write; the operation never executes.
- The fragment is recorded `REJECTED` with the guardrail reason in its status chip.
- The block's `pendingFragmentCount` is not incremented for the rejected fragment.
- The writer loop continues its next cycle without crashing.

## J4 — Sanitizer: special-category attribute redacted

**Preconditions:** As J1. Under the mock LLM, the `writer-agent.json` set includes an entry whose content contains a synthetic personal identifier (e.g., `contact: jane.doe@example.com`); with a real model, a seed whose topic naturally surfaces contact information.

**Steps:**
1. Seed a block; watch the fragment from the writer that produces the personal-identifier content.

**Expected:**
- The sanitizer (S1) fires after the agent returns; the identifier is replaced with a typed placeholder (e.g., `[REDACTED:EMAIL]`) before the fragment is committed.
- The fragment is recorded with `status: SANITIZED` and a non-empty `sanitizedNote` listing the redacted field type.
- The `sanitizedNote` badge is visible in the fragment panel.
- The consolidated snapshot, if this fragment is later merged, does not contain the raw identifier.

## J5 — On-demand consolidation

**Preconditions:** As J1, with at least one `ACTIVE` block that has one or more `PENDING` fragments (fragment count below the automatic threshold).

**Steps:**
1. Click the `Consolidate` button on the block card (or `POST /api/blocks/{id}/consolidate`).
2. Watch the block.

**Expected:**
- The block moves immediately to `CONSOLIDATING`; new write calls are refused.
- `ConsolidationWorkflow` runs `ConsolidatorAgent`; a new `ConsolidatedSnapshot` is committed.
- The block moves to `CONSOLIDATED`; the board card shows the updated consolidation summary and a refreshed `lastConsolidatedAt` timestamp.
- All previously `PENDING` (and `SANITIZED`) fragments for the block move to `MERGED`.
- `GET /api/blocks/{id}` returns the new snapshot content.
