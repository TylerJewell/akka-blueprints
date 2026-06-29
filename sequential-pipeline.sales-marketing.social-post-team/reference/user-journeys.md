# User journeys

Acceptance journeys. Each passes when its expected state changes and UI updates occur.

## J1 — Submit a concept and watch it compose
- **Preconditions:** service running; App UI tab open.
- **Steps:** type a concept, click Submit (`POST /api/concepts`).
- **Expected:** response returns `{ postId }`. Via SSE the post appears in `RESEARCHING`, moves to `COMPOSING`, then to `AWAITING_APPROVAL` with a non-empty `caption`, non-empty `hashtags`, and non-empty `visualBrief`, typically within ~30s. Approve/Reject buttons appear.

## J2 — Approve and publish
- **Preconditions:** a post in `AWAITING_APPROVAL` (from J1).
- **Steps:** click Approve (`POST /api/posts/{id}/approve` with `{ approvedBy, comment }`).
- **Expected:** post transitions to `APPROVED` then `PUBLISHED` with a non-null `publishedPermalink`. The status pill turns green; Approve/Reject buttons disappear.

## J3 — Reject a post
- **Preconditions:** a post in `AWAITING_APPROVAL`.
- **Steps:** click Reject, enter a reason, submit (`POST /api/posts/{id}/reject`).
- **Expected:** post transitions to `REJECTED`; `rejectReason` shows in the card; the workflow ends.

## J4 — Escalate a stale post
- **Preconditions:** a post in `AWAITING_APPROVAL`; no human action.
- **Steps:** wait past the 2-minute threshold.
- **Expected:** `StalePostMonitor` transitions the post to `ESCALATED`; the status pill turns red; Approve/Reject buttons disappear.

## J5 — Background load from the simulator
- **Preconditions:** service running; no UI interaction.
- **Steps:** wait.
- **Expected:** `ConceptSimulator` enqueues a fresh concept every 30s from `sample-events/concepts.jsonl`; `ConceptConsumer` starts a workflow per concept; new posts appear in the list on their own.

## J6 — URL guard blocks an off-allowlist fetch
- **Preconditions:** service running.
- **Steps:** during research, the agent (or a direct `POST /api/tools/browse`) requests a host outside the allowlist.
- **Expected:** the before-tool-call guard returns `403`; the fetch does not run; the research step proceeds using an allowlisted source.
