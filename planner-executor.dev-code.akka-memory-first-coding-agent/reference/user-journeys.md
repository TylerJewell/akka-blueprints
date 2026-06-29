# User journeys — akka-memory-first-coding-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Init happy path: memory blocks built and system prompt rewritten

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`). Fixture codebase present at `src/main/resources/sample-data/codebase/`.

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. The `ResearchSimulator` fires on startup — a project card appears with status `RESEARCHING`.

**Expected:**
- Within 5 s of startup, a project card for the sample project appears with status `RESEARCHING` via SSE.
- Within ~3 minutes, status transitions to `READY`.
- The expanded project card shows:
  - A research plan with 4–8 files in `filesToRead` and 3–6 questions.
  - 4–6 memory blocks, each with a non-empty `name`, `content`, and `sourceFiles`.
  - A system prompt field containing the rewritten 2–3 sentence prompt (different from the static `prompts/memory-writer.md` content, referencing the actual project's framework and package).
- Subsequent `/init` calls on the same project path return `202` with the same `projectId` (idempotency dedup within 30 s).

## J2 — Guardrail blocks an out-of-scope DELETE

**Preconditions:** A project in `READY` state from J1.

**Steps:**
1. Submit the edit instruction: "Delete the legacy configuration file at `/etc/legacy.conf` as it is no longer needed."

**Expected:**
- `EditPlannerAgent` produces a `PatchPlan` whose first edit has `kind = DELETE` and `filePath = "/etc/legacy.conf"`.
- `EditGuardrail.vet` rejects the edit because `/etc/legacy.conf` is outside `/workspace/` and does not match the allowed delete patterns.
- A `PatchBlocked` entry appears on the session's patch log with `verdict = BLOCKED_BY_GUARDRAIL` and `blocker` explaining the path violation.
- `EditPlannerAgent` receives the blocker and revises the plan. The revised plan either skips the delete or proposes an alternative path under `/workspace/`.
- The session either completes via the revised plan or ends in `FAILED` after the blocker budget is exhausted. The original `/etc/legacy.conf` delete never executes.

## J3 — Test gate ends session on failing tests

**Preconditions:** A project in `READY` state. The fixture test result for the project is configured to return `passed=false, failed=2` (see `sample-data/test-results/`).

**Steps:**
1. Submit any edit instruction for the project that has a failing test fixture.
2. The workflow applies the first patch successfully.

**Expected:**
- After `PatchApplied`, `FixtureTestRunner.run` returns `passed=false, total=12, failed=2` for this project.
- The workflow emits `TestsFailed` on `EditSessionEntity`.
- Session status moves to `TESTS_FAILED`.
- The UI's expanded session card shows the test output block with `failed: 2` and the fixture output text.
- `ProjectEntity.memory.blocks` is not updated — the failed patch is not promoted to project memory.

## J4 — Operator halt drains the in-flight apply gracefully

**Preconditions:** A project in `READY` state. Edit instruction produces a plan with 3 or more patches.

**Steps:**
1. Submit an edit instruction for the project.
2. While session status is `APPLYING` and the first patch is in flight, click **Halt destructive edits** in the operator pane and enter a reason.

**Expected:**
- The in-flight `PatchApplied` and `TestsPassed` (or `TestsFailed`) events are recorded normally.
- The next loop iteration reads `SystemControlEntity.halted = true` in `checkHaltStep`.
- The workflow emits `SessionHalted` and ends the session.
- Session status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane shows the `HALTED` pill in real time via the `control-update` SSE event.
- Clicking **Resume** calls `POST /api/control/resume`; the pane reverts to the `Halt` button. New edit sessions started after resume proceed normally.

## J5 — Stale session is auto-timed-out

**Preconditions:** Service running with `stuck.threshold-minutes = 1` test override set in `application.conf`.

**Steps:**
1. Submit any edit instruction.
2. Configure the `EditPlannerAgent` mock response to return a plan with a single patch that always returns `ok=false` with no revision path, causing the session to sit in `APPLYING` without progress.

**Expected:**
- After 1 minute in `APPLYING`, `StaleSessionMonitor` calls `EditSessionEntity.timeoutSession`.
- Session status moves to `TIMED_OUT`.
- `failureReason` contains `"timed out: no progress after 1m"`.
- The workflow's `decideStep` reads `TIMED_OUT` on the entity and exits cleanly.
