# User journeys — gitwiki

Acceptance criteria. Each journey defines a precondition set, a sequence of steps, and the expected outcome the acceptance bar requires.

---

## J1 — Content addition pushed successfully

**Precondition:** Service is running. `GITHUB_TOKEN` is set and has `repo` scope on the target repository. `akka.gitwiki.target-branch = "wiki-drafts"` and `akka.gitwiki.allowed-namespace = "wiki/"` are in `application.conf`. Repository exists with a `wiki-drafts` branch. One seeded page `wiki/getting-started.md` is in the branch.

**Steps:**
1. Open the App UI tab. Select `wiki/getting-started.md` from the Page dropdown.
2. In Requested changes type: `Add a Prerequisites section before Installation listing Java 21 and Maven 3.9.`
3. Fill Author: `alice-editor`. Click **Submit update**.
4. Observe the card appear as `SUBMITTED` in the live list.
5. Within ~5 s the card transitions to `EDITING`. Within ~30 s it transitions to `COMMIT_READY`, then `PUSH_IN_PROGRESS`, then `PUSHED`.
6. The commit SHA chip appears on the card. The right pane shows the diff summary and the commit message.

**Expected outcome:**
- The update card reaches `PUSHED` within 60 s.
- The commit SHA links to a real commit on the target repository's `wiki-drafts` branch.
- The commit message on GitHub is ≤ 72 characters, imperative mood, and describes the Prerequisites addition.
- The Prerequisites section appears in the committed file between the title and the Installation section.
- The `PageEntity` state contains `pushResult.status = SUCCESS` and a non-null `commitSha`.

---

## J2 — Push blocked by guardrail (wrong branch)

**Precondition:** Service is running. `GITHUB_TOKEN` is valid. `akka.gitwiki.target-branch = "wiki-drafts"` in config. An actor has misconfigured the agent to push to `main` instead of `wiki-drafts`.

**Steps:**
1. Submit an update to `wiki/architecture-overview.md` with Author `bob-reviewer`.
2. The workflow starts normally; the agent produces a valid `CommitDraft`.
3. When `GitPushConsumer` calls the guardrail, the target ref is `main` — not `wiki-drafts`.

**Expected outcome:**
- `GitPushGuardrail.check(...)` returns `Guardrail.reject("target branch 'main' is protected; configured branch is 'wiki-drafts'")`.
- `GitPushConsumer` calls `PageEntity.recordPushResult(PushResult.rejected(...))`.
- The card transitions to `PUSH_REJECTED`.
- The right pane shows the rejection reason in a red callout: `"target branch 'main' is protected; configured branch is 'wiki-drafts'"`.
- No commit is created on the repository.

---

## J3 — Startup fails without GitHub token

**Precondition:** `GITHUB_TOKEN` is not set in the environment. `application.conf` has `akka.gitwiki.target-branch`, `akka.gitwiki.repo-owner`, and `akka.gitwiki.repo-name` set to real values.

**Steps:**
1. Run `/akka:build`.
2. Observe the service startup log.

**Expected outcome:**
- The JVM exits with code 1 within 5 s of startup.
- The log contains the exact message: `StartupConfigGate: GITHUB_TOKEN is not set. Set it as an environment variable or provide it via /akka:specify.`
- The service does not bind to port 9821 and does not accept any HTTP requests.
- No partial state is written to any entity.

---

## J4 — Concurrent updates produce a conflict

**Precondition:** Service is running with a valid token. `wiki/api-reference.md` exists on `wiki-drafts`.

**Steps:**
1. Submit update A to `wiki/api-reference.md` (add an Errors section).
2. Immediately submit update B to `wiki/api-reference.md` (fix a broken link).
3. Update A finishes and pushes its commit to `wiki-drafts` first.
4. Update B's `GitPushConsumer` calls the GitHub ref-update API while A's commit is the branch tip — B's push is based on the original tip, creating a non-fast-forward condition.

**Expected outcome:**
- Update A's card reaches `PUSHED` with a valid commit SHA.
- Update B's card reaches `CONFLICT`.
- The right pane for update B shows the conflicting commit SHA (A's SHA) and a note: "rebase manually against wiki-drafts before resubmitting."
- `PageEntity` for update B records `pushResult.status = CONFLICT` and the conflicting SHA.
- The `wiki/api-reference.md` file on GitHub contains only A's changes (B's were not force-pushed).

---

## J5 — Agent applies changes and produces a valid commit message

**Precondition:** Service is running. Seeded page `wiki/api-reference.md` contains a broken markdown link `[rate limits](/docs/limits)`.

**Steps:**
1. Submit an update: Requested changes = `Fix the broken link in the See Also section: /docs/limits should be /docs/rate-limits.` Author = `charlie-tech-writer`.
2. Wait for the card to reach `PUSHED`.

**Expected outcome:**
- The `CommitDraft.commitMessage` is ≤ 72 characters and starts with an imperative verb (e.g., `Fix broken rate-limits link in See Also`).
- The `CommitDraft.diffSummary` is one sentence in past tense referencing the link fix.
- The pushed file on GitHub contains `[rate limits](/docs/rate-limits)` in the See Also section.
- The `before-tool-call` guardrail was not triggered (push proceeded on the first attempt).

---

## J6 — Author mismatch blocked by guardrail

**Precondition:** Service is running. `akka.gitwiki.target-branch = "wiki-drafts"`. The agent is triggered with `author = "external-contributor"` but the guardrail's configured author allowlist does not include that value. (For this journey the allowlist feature is enabled by the deployer via `akka.gitwiki.allowed-authors`.)

**Steps:**
1. Submit an update to `wiki/getting-started.md` with Author `external-contributor`.
2. The workflow proceeds; the agent produces a `CommitDraft`.
3. `GitPushConsumer` calls the guardrail with `author = "external-contributor"`.

**Expected outcome:**
- `GitPushGuardrail` returns `Guardrail.reject("author 'external-contributor' is not in the allowed-authors list")`.
- The card transitions to `PUSH_REJECTED` with that rejection reason in the right pane.
- No commit is created on the repository.
