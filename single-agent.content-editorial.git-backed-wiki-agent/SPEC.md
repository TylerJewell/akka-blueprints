# SPEC — gitwiki

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GitWiki Demo Agent.
**One-line pitch:** A user submits a wiki page update; one AI agent applies the edit and commits and pushes it to GitHub, with every `git push` gated by a `before-tool-call` guardrail and a configuration gate that attests the repo token and target branch at startup.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One `WikiEditorAgent` (AutonomousAgent) carries the entire edit-and-commit decision; the surrounding components validate its environment, route events, and audit its output. Two governance mechanisms are wired around the agent:

- A **configuration gate** runs at service startup and verifies that `GITHUB_TOKEN` is non-empty, the `target_branch` is declared in configuration, and the token has `repo` scope (probed by a dry-run API call). If any check fails, the service refuses to start and emits a clear error message naming the missing requirement.
- A **before-tool-call guardrail** intercepts every `git push` invocation the agent would issue. It verifies that the target ref matches the configured `target_branch`, that the author matches the submitting user, and that the page path is within the allowed namespace. A push that fails any check is rejected before the tool executes; the rejection is recorded in the entity.

The blueprint shows that external-write tool calls — git pushes — need explicit pre-flight checks beyond what the model chooses to do on its own.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **page** from a dropdown (three seeded wiki pages are available: `getting-started.md`, `architecture-overview.md`, `api-reference.md`) or types a new page path.
2. The user edits the page body in a textarea. A diff preview appears showing insertions and deletions against the current stored content.
3. The user fills in **Author** (a display name) and clicks **Submit update**. The UI POSTs to `/api/updates` and receives an `updateId`.
4. The update card appears in the live list as `SUBMITTED`. Within ~1 s it transitions to `EDITING` as the workflow starts.
5. Within ~10–30 s, the agent's edit step completes. The card transitions to `COMMIT_READY`. A diff summary is visible in the card detail.
6. The `GitPushConsumer` fires. The guardrail runs; on a clean check the push proceeds. The card transitions to `PUSH_IN_PROGRESS` then `PUSHED`. The commit SHA and a link to the GitHub commit appear on the card.
7. If the guardrail blocks the push, the card transitions to `PUSH_REJECTED` with the rejection reason visible.
8. The user can submit another update; the live list keeps the history visible, newest first.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WikiEndpoint` | `HttpEndpoint` | `/api/updates/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PageEntity`, `PageView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PageEntity` | `EventSourcedEntity` | Per-update lifecycle: submitted → editing → commit-ready → push-in-progress → pushed / push-rejected / failed. Source of truth. | `WikiEndpoint`, `GitPushConsumer`, `WikiUpdateWorkflow` | `PageView` |
| `GitPushConsumer` | `Consumer` | Subscribes to `CommitReady` events; runs `GitPushGuardrail`; executes the push; calls `PageEntity.recordPushResult`. | `PageEntity` events | `PageEntity` |
| `WikiUpdateWorkflow` | `Workflow` | One workflow per updateId. Steps: `configCheckStep` → `editStep` → `awaitPushStep`. | started by `WikiEndpoint` on submit | `WikiEditorAgent`, `PageEntity` |
| `WikiEditorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the page title, current body, and requested changes as a task attachment; returns `CommitDraft` (edited body + commit message). | invoked by `WikiUpdateWorkflow` | returns draft |
| `GitPushGuardrail` | supporting class | `before-tool-call` hook: validates target ref, author, and page namespace before any push executes. | invoked by `GitPushConsumer` | blocks or passes |
| `StartupConfigGate` | supporting class | Configuration gate: runs at Bootstrap; verifies `GITHUB_TOKEN`, `target_branch`, and repo access. Fails fast on missing config. | Bootstrap | none |
| `PageView` | `View` | Read model: one row per update for the UI. | `PageEntity` events | `WikiEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PageUpdateRequest(
    String updateId,
    String pagePath,
    String pageTitle,
    String currentBody,
    String requestedChanges,
    String author,
    Instant submittedAt
) {}

record CommitDraft(
    String editedBody,
    String commitMessage,
    String diffSummary        // short human-readable description of changes
) {}

record PushResult(
    PushStatus status,
    String commitSha,           // non-null on success
    String rejectionReason,     // non-null on rejection
    Instant pushedAt
) {}
enum PushStatus { SUCCESS, REJECTED, CONFLICT, FAILED }

record CommitOutcome(
    String updateId,
    Optional<CommitDraft> draft,
    Optional<PushResult> pushResult,
    UpdateStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum UpdateStatus {
    SUBMITTED, EDITING, COMMIT_READY, PUSH_IN_PROGRESS, PUSHED, PUSH_REJECTED, CONFLICT, FAILED
}
```

Events on `PageEntity`: `UpdateSubmitted`, `EditingStarted`, `CommitReady`, `PushStarted`, `PushCompleted`, `PushRejected`, `UpdateFailed`.

Every nullable lifecycle field on `CommitOutcome` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/updates` — body `{ pagePath, pageTitle, currentBody, requestedChanges, author }` → `{ updateId }`.
- `GET /api/updates` — list all updates, newest-first.
- `GET /api/updates/{id}` — one update.
- `GET /api/updates/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: GitWiki Demo Agent</title>`.

The App UI tab is a two-column layout: a left column with the update submission form and the live list of update cards (status pill + commit SHA chip + age), and a right pane with the selected update's detail — diff preview, agent-generated commit message, push result, and rejection reason if applicable.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`, registered on `WikiEditorAgent`): intercepts every `git push` tool call the agent would issue. Checks that (1) the target ref equals the configured `target_branch`, (2) the author field on the commit matches the submitting user from `PageUpdateRequest.author`, and (3) the `pagePath` is within the configured allowed namespace (default: any path under `wiki/`). On any failure returns a structured rejection; the push does not execute; `GitPushConsumer` calls `PageEntity.recordPushResult(PushResult.rejected(reason))`.
- **H1 — configuration gate** (`ci-gate`, `configuration-gate`, runs at Bootstrap via `StartupConfigGate`): at service startup verifies that `GITHUB_TOKEN` is non-empty, that `akka.gitwiki.target-branch` is set in `application.conf` or environment, and that the token can reach the repository (a `/repos/{owner}/{repo}` probe). If any check fails, `StartupConfigGate.verify()` throws, Bootstrap rethrows, and the JVM exits with exit code 1 and a message naming the failed check.

## 9. Agent prompts

- `WikiEditorAgent` → `prompts/wiki-editor.md`. The single decision-making LLM. System prompt instructs it to apply the requested changes to the current page body while preserving markdown structure, then produce a `CommitDraft` with the edited body, a concise commit message (≤ 72 chars), and a one-sentence diff summary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a content addition to `getting-started.md`; within 30 s the commit SHA appears in the UI and the page on GitHub reflects the new content.
2. **J2** — User submits an update targeting a protected branch (`main`); the guardrail blocks the push; the card shows `PUSH_REJECTED` with the reason "target branch 'main' is protected; configured branch is 'wiki-drafts'".
3. **J3** — Service starts without `GITHUB_TOKEN` set; startup fails within 2 s with exit code 1 and a message: "StartupConfigGate: GITHUB_TOKEN is not set. Set it as an environment variable or provide it via /akka:specify.".
4. **J4** — Two concurrent updates to `architecture-overview.md` submit within 1 s of each other; the second update's push detects a non-fast-forward condition; that card transitions to `CONFLICT` with the conflicting commit SHA visible.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named gitwikidemoagent demonstrating the single-agent × content-editorial cell.
Requires GITHUB_TOKEN env var (configuration gate enforced at Bootstrap). Maven group
io.akka.samples. Maven artifact single-agent-content-editorial-git-backed-wiki-agent. Java
package io.akka.samples.gitwikidemoagent. Akka 3.6.0. HTTP port 9821.

Components to wire (exactly):

- 1 AutonomousAgent WikiEditorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/wiki-editor.md>) and
  .capability(TaskAcceptance.of(EDIT_PAGE).maxIterationsPerTask(2)). The task receives the
  requested changes as its instruction text and the current page body as a task ATTACHMENT
  (NOT as inline prompt text — TaskDef.attachment("current-body.md", bodyBytes) is the
  canonical call). Output: CommitDraft{editedBody: String, commitMessage: String,
  diffSummary: String}. The agent is configured with a before-tool-call guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent call returns an error outcome; the consumer records
  PushResult.rejected(reason) on the entity.

- 1 Workflow WikiUpdateWorkflow per updateId with three steps:
  * configCheckStep — calls StartupConfigGate.isHealthy(); if false, transitions entity to
    FAILED immediately. WorkflowSettings.stepTimeout 5s.
  * editStep — emits EditingStarted, then calls componentClient.forAutonomousAgent(
    WikiEditorAgent.class, "editor-" + updateId).runSingleTask(
      TaskDef.instructions(formatChanges(update.requestedChanges))
        .attachment("current-body.md", update.currentBody.getBytes())
    ) — fetches CommitDraft result, calls PageEntity.markCommitReady(draft).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(1)
    .failoverTo(WikiUpdateWorkflow::error).
  * awaitPushStep — polls PageEntity.getUpdate every 1s until status is PUSHED,
    PUSH_REJECTED, CONFLICT, or FAILED. WorkflowSettings.stepTimeout 30s. error step
    transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PageEntity (one per updateId). State CommitOutcome{updateId: String,
  draft: Optional<CommitDraft>, pushResult: Optional<PushResult>, status: UpdateStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. UpdateStatus enum: SUBMITTED,
  EDITING, COMMIT_READY, PUSH_IN_PROGRESS, PUSHED, PUSH_REJECTED, CONFLICT, FAILED.
  Events: UpdateSubmitted{request: PageUpdateRequest}, EditingStarted{},
  CommitReady{draft: CommitDraft}, PushStarted{}, PushCompleted{sha: String, pushedAt: Instant},
  PushRejected{reason: String, pushedAt: Instant}, UpdateFailed{reason: String}.
  Commands: submit, markEditing, markCommitReady, markPushStarted, recordPushResult, fail,
  getUpdate. emptyState() returns CommitOutcome.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer GitPushConsumer subscribed to PageEntity events; on CommitReady runs
  GitPushGuardrail.check(pagePath, targetRef, author) — if rejected, calls
  PageEntity.recordPushResult(PushResult.rejected(reason)) and returns; if passed, calls
  PageEntity.markPushStarted(), executes the actual GitHub API push (POST /repos/{owner}/{repo}/
  git/commits then PATCH /repos/{owner}/{repo}/git/refs/heads/{branch} via JDK HttpClient with
  Authorization: token GITHUB_TOKEN), and calls PageEntity.recordPushResult with the result.
  Handles 422 (non-fast-forward → CONFLICT) and non-2xx separately.

- 1 View PageView with row type UpdateRow (mirrors CommitOutcome, carries pagePath and author
  from the UpdateSubmitted event payload). Table updater consumes PageEntity events. ONE query
  getAllUpdates: SELECT * AS updates FROM page_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * WikiEndpoint at /api with POST /updates (body {pagePath, pageTitle, currentBody,
    requestedChanges, author}; mints updateId; calls PageEntity.submit; starts
    WikiUpdateWorkflow; returns {updateId}), GET /updates (list from getAllUpdates,
    sorted newest-first), GET /updates/{id} (one row), GET /updates/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Supporting classes:

- WikiTasks.java declaring one Task<R> constant: EDIT_PAGE = Task.name("Edit wiki page")
  .description("Apply the requested changes to the current page body and produce a CommitDraft")
  .resultConformsTo(CommitDraft.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records PageUpdateRequest, CommitDraft, PushResult, PushStatus, CommitOutcome,
  UpdateStatus.

- GitPushGuardrail.java implementing the before-tool-call hook. Reads the configured
  target_branch and allowed_namespace from application.conf. Checks that (1) the target ref
  equals target_branch, (2) the author matches the request author, and (3) the pagePath starts
  with the allowed namespace. Returns Guardrail.reject(reason) on failure, Guardrail.pass()
  on success.

- StartupConfigGate.java — called from Bootstrap.java before the Akka runtime starts.
  Reads GITHUB_TOKEN from the environment. Reads akka.gitwiki.target-branch and
  akka.gitwiki.repo-owner and akka.gitwiki.repo-name from config. Calls
  GET https://api.github.com/repos/{owner}/{repo} with Authorization: token {GITHUB_TOKEN}
  to confirm access. Throws StartupConfigException with a human-readable message if any
  check fails. Bootstrap catches and re-throws, causing the JVM to exit with code 1.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9821,
  akka.gitwiki.target-branch = "wiki-drafts", akka.gitwiki.repo-owner = "TO_BE_SET",
  akka.gitwiki.repo-name = "TO_BE_SET", akka.gitwiki.allowed-namespace = "wiki/",
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. The WikiEditorAgent.definition() binds the configured
  provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-pages.jsonl with 3 seeded wiki pages:
  getting-started.md (600 words), architecture-overview.md (900 words),
  api-reference.md (500 words). Each is plausible technical documentation that the agent
  can meaningfully edit.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root with purpose.primary_function = wiki-content-management,
  data.data_classes.pii = false, decisions.authority_level = autonomous
  (the agent executes the write without human approval), oversight.human_in_loop = false,
  oversight.human_on_loop = true (a human can review the commit on GitHub before merge),
  failure.failure_modes including "unwanted-overwrite", "push-to-wrong-branch",
  "missing-repo-token", "merge-conflict"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/wiki-editor.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: GitWiki Demo Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of update cards; right = selected update detail with diff
  preview, commit message, push result). Browser title exactly:
  <title>Akka Sample: GitWiki Demo Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct CommitDraft outputs per EDIT_PAGE task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/<task-id>.json,
  picks one entry pseudo-randomly per call (seedFor(updateId)), and deserialises into the
  task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    edit-page.json — 6 CommitDraft entries covering realistic page edits (additions,
      rewrites, deletions). Each entry has a non-empty editedBody, a concise commitMessage
      (≤ 72 chars), and a one-sentence diffSummary. Plus 1 entry with an intentionally
      oversized commitMessage (> 72 chars) and 1 with an empty editedBody — used to exercise
      error paths. The mock selects the oversized entry on the second update of every three
      (modulo seed) so the guardrail's author check and the length validation have test
      coverage.
- A MockModelProvider.seedFor(updateId) helper makes per-update selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WikiEditorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion WikiTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (configCheckStep 5s, editStep
  60s, awaitPushStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on CommitOutcome is Optional<T>.
- Lesson 7: WikiTasks.java with EDIT_PAGE = Task.name(...).description(...)
  .resultConformsTo(CommitDraft.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9821 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Requires GitHub token" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WikiEditorAgent).
  GitPushGuardrail and StartupConfigGate are plain Java classes, not agents.
- The page body is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated editStep uses TaskDef.attachment(...) and not string interpolation.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
