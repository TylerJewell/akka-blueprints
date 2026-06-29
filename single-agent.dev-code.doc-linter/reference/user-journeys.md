# User journeys — doc-linter

Acceptance journeys. Each journey's "done" condition is the bar `/akka:specify` output must
pass.

## J1 — Lint a file via API and read findings

- **Preconditions:** service running on `http://localhost:9251/`; sample-data files present.
- **Steps:** POST `/api/lint` with `{ "filePath": "sample-data/getting-started.md" }`.
- **Expected:** response carries a `runId`. The run appears in `REQUESTED`, then `LINTED`
  within ~30 s with a non-empty `findings` list and a non-null `summary`.
- **Done:** `GET /api/lint/{runId}` returns `status: LINTED` with findings and summary.

## J2 — Read the documentation gate

- **Preconditions:** a known-bad sample file (seeded `ERROR` finding) and a clean sample
  file exist.
- **Steps:** lint the bad file, then the clean file.
- **Expected:** the bad file's run shows `gate: FAIL`; the clean file's run shows `gate: PASS`.
- **Done:** the App UI gate badge reads FAIL then PASS; the integration test asserts both.

## J3 — Block a path traversal

- **Preconditions:** service running.
- **Steps:** POST `/api/lint` with `{ "filePath": "../../etc/passwd" }`.
- **Expected:** the run is recorded `BLOCKED` with a `blockReason`; no file outside the
  sample-data root is read; `findings` is empty.
- **Done:** `GET /api/lint/{runId}` returns `status: BLOCKED` and a non-null `blockReason`.

## J4 — Apply feedback and shift rule emphasis

- **Preconditions:** a `LINTED` run with at least one finding.
- **Steps:** POST `/api/lint/{id}/feedback` with `{ "rule": "line-length", "note": "keep flagging" }`.
- **Expected:** the run moves to `FEEDBACK_APPLIED`; `FeedbackConsumer` bumps the
  `line-length` weight in `RuleProfileEntity`; the next lint of a file with that finding
  ranks it higher in the summary.
- **Done:** `RuleProfileEntity.getProfile` shows the increased weight and a subsequent run's
  `orderedFindingRules` reflects it.

## J5 — Background load from the simulator

- **Preconditions:** service running; no manual interaction.
- **Steps:** wait one simulator tick (~30 s).
- **Expected:** `LintSimulator` seeds a lint run from the next path in
  `sample-data/manifest.txt`; the run list is non-empty without any user action.
- **Done:** `GET /api/lint` returns at least one run after a tick.
