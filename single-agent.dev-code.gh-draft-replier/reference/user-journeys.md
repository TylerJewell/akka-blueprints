# User journeys — gitty

## J1 — Queue a seeded issue and receive a draft

**Preconditions:** Service running on `http://localhost:9510/`; a valid model-provider API key set (or mock LLM selected at scaffold time); `GITHUB_TOKEN` set with `repo` scope (or mock thread fetcher active).

**Steps:**
1. Open `http://localhost:9510/` → App UI tab.
2. Click **Load seeded example** to fill the Thread URL and Context note from the first seeded thread (a feature request left unanswered for 90 days).
3. Fill in `Queued by` with any identifier (e.g., `maintainer-01`).
4. Click **Queue draft**.

**Expected:**
- The new card appears in the live list with status `QUEUED` within 1 s.
- The card transitions to `THREAD_FETCHED` within 2 s. The right-pane detail shows the thread title, comment count, and fetched timestamp.
- Within 30 s the card reaches `PENDING_REVIEW`. The right pane shows: a draft text of ≤ 800 characters, a `tone` badge in one of the four allowed values, and a references list.
- The **Approve** and **Discard** buttons are visible.

## J2 — Guardrail blocks a non-compliant draft

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `draft-reply.json` includes deliberately non-compliant entries (a confrontational phrase; a tone value outside the enum).

**Steps:**
1. Queue any seeded thread three times in a row (J1 steps × 3).
2. Watch the third draft's lifecycle in the browser dev tools network panel (`/api/drafts/sse`).

**Expected:**
- The third draft's first agent iteration produces a non-compliant `DraftReply`.
- The `before-agent-response` guardrail rejects it. The non-compliant text NEVER lands in `DraftEntity` — there is no `DraftProduced` event with the non-compliant payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a compliant draft. The card transitions to `PENDING_REVIEW` with a draft that passes all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Maintainer approves a draft (with edits)

**Preconditions:** J1 completed; a draft is in `PENDING_REVIEW`.

**Steps:**
1. In the right pane, edit the draft text (e.g., add "cc @original-reporter").
2. Click **Approve**.

**Expected:**
- The card status transitions to `APPROVED` within 1 s.
- The right pane switches from edit mode to readonly mode. The displayed text matches the edited version, not the agent's original.
- A **Copy to clipboard** button appears. Clicking it copies the approved text.
- Fetching `GET /api/drafts/{id}` returns `action.editedText` equal to the edited text and `status = APPROVED`.
- No GitHub POST is made by the system.

## J4 — Maintainer discards a draft

**Preconditions:** J1 completed; a draft is in `PENDING_REVIEW`.

**Steps:**
1. Click **Discard** on the selected card.

**Expected:**
- The card status transitions to `DISCARDED` within 1 s.
- The right pane shows a muted "Discarded by {actorId}" note. The Approve and Discard buttons disappear.
- The card remains visible in the live list (it does not disappear). Its status pill shows `DISCARDED` in muted colour.
- Fetching `GET /api/drafts/{id}` returns `status = DISCARDED` and `finishedAt` populated.

## J5 — GitHub fetch failure transitions to FAILED

**Preconditions:** Service running with a real GitHub token. Use a URL pointing to a private repository the token cannot access.

**Steps:**
1. Paste a GitHub URL for an inaccessible repository.
2. Fill in `Queued by` and click **Queue draft**.

**Expected:**
- The card appears in `QUEUED` state.
- Within 5 s (before the workflow step times out) the card transitions to `FAILED`. The right pane shows the failure reason (e.g., "GitHub API returned 404").
- The entity retains the original `QueueRequest` for audit — the maintainer can inspect what was submitted.
- No draft text is produced.
