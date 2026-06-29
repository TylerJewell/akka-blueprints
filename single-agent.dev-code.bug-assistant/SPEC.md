# SPEC — bug-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BugAssistant.
**One-line pitch:** A user submits a bug report; one AI agent searches ticketing data and web sources for context (each search is a governed tool call), proposes a resolution, and writes the result back to the ticket — gated by a `before-tool-call` guardrail that prevents malformed or incomplete ticket writes from executing.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `BugResolutionAgent` (AutonomousAgent) carries the entire investigation and resolution decision; the surrounding components only prepare its input, stream its state, and audit its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** intercepts every `write_ticket` tool call the agent issues. It asserts the resolution body is non-empty, the proposed status is in the allowed enum, and the ticket id in the tool arguments matches the bug currently being resolved. A rejected call is returned to the agent loop as a structured error; the agent must correct and retry within its iteration budget.

The blueprint shows that even a straightforward "look it up and fix it" automation benefits from a write-gate: the guardrail stops the agent from accidentally closing the wrong ticket or submitting a one-word resolution that would confuse a human reviewer.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a bug title and description in the **Bug Report** form (or picks one of three seeded examples — a Java NullPointerException, a Go channel deadlock, a Python async race condition).
2. The user optionally tags the bug with a `priority` (LOW / MEDIUM / HIGH / CRITICAL) and a `component` label, then clicks **Submit bug**.
3. The UI POSTs to `/api/bugs` and receives a `bugId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `ENRICHED` — initial ticket metadata (reporter, labels, timestamp) is visible in the card detail.
5. Within ~10–30 s, the agent finishes. The card transitions through `INVESTIGATING` → `RESOLVED`. The resolution appears: a `ResolutionStatus` badge (FIXED / NEEDS_MORE_INFO / WONT_FIX / DUPLICATE), a `resolutionBody` paragraph, a `confidenceLevel` chip (HIGH / MEDIUM / LOW), and the list of tool calls the agent made (search queries + results, ticket reads, the final write).
6. The user can submit another bug; the live list keeps all history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BugEndpoint` | `HttpEndpoint` | `/api/bugs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BugEntity`, `BugView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BugEntity` | `EventSourcedEntity` | Per-bug lifecycle: submitted → enriched → investigating → resolved or failed. Source of truth. | `BugEndpoint`, `TicketSyncConsumer`, `BugWorkflow` | `BugView` |
| `TicketSyncConsumer` | `Consumer` | Subscribes to `BugSubmitted` events; fetches simulated ticket metadata; calls `BugEntity.attachEnriched`. | `BugEntity` events | `BugEntity` |
| `BugWorkflow` | `Workflow` | One workflow per bug. Steps: `awaitEnrichedStep` → `investigateStep` → `closeStep`. | started by `TicketSyncConsumer` once enriched event lands | `BugResolutionAgent`, `BugEntity` |
| `BugResolutionAgent` | `AutonomousAgent` | The one decision-making LLM. Receives bug details in the task definition; uses `search_web` and `read_ticket` tools to gather context; issues a `write_ticket` tool call gated by the guardrail. | invoked by `BugWorkflow` | returns `Resolution` |
| `TicketWriteGuardrail` | supporting class | `before-tool-call` hook registered on `BugResolutionAgent`. Validates every `write_ticket` call before it executes. | invoked by agent loop | pass or reject |
| `BugView` | `View` | Read model: one row per bug for the UI. | `BugEntity` events | `BugEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BugReport(
    String bugId,
    String title,
    String description,
    Priority priority,
    String component,
    String reportedBy,
    Instant reportedAt
) {}
enum Priority { LOW, MEDIUM, HIGH, CRITICAL }

record TicketMetadata(
    String ticketKey,        // e.g. "PROJ-1042"
    String assignee,
    List<String> labels,
    String project,
    Instant fetchedAt
) {}

record SearchResult(
    String query,
    String source,           // "web" | "ticket-history"
    String snippet,
    String url
) {}

record Resolution(
    ResolutionStatus status,
    String resolutionBody,
    ConfidenceLevel confidenceLevel,
    List<SearchResult> evidence,
    Instant resolvedAt
) {}
enum ResolutionStatus { FIXED, NEEDS_MORE_INFO, WONT_FIX, DUPLICATE }
enum ConfidenceLevel { HIGH, MEDIUM, LOW }

record Bug(
    String bugId,
    Optional<BugReport> report,
    Optional<TicketMetadata> ticket,
    Optional<Resolution> resolution,
    BugStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BugStatus {
    SUBMITTED, ENRICHED, INVESTIGATING, RESOLVED, FAILED
}
```

Events on `BugEntity`: `BugSubmitted`, `TicketEnriched`, `InvestigationStarted`, `ResolutionRecorded`, `BugFailed`.

Every nullable lifecycle field on the `Bug` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/bugs` — body `{ title, description, priority, component, reportedBy }` → `{ bugId }`.
- `GET /api/bugs` — list all bugs, newest-first.
- `GET /api/bugs/{id}` — one bug.
- `GET /api/bugs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: BugAssistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted bugs (status pill + resolution badge + priority chip + age) and a right pane with the selected bug's detail — report description, ticket metadata, tool-call trace, resolution body, confidence chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `write_ticket` tool call the agent issues. Asserts (1) `resolutionBody` is non-empty and at least 20 characters, (2) `status` is in `{FIXED, NEEDS_MORE_INFO, WONT_FIX, DUPLICATE}`, (3) `ticketId` in the tool arguments equals the `bugId` the workflow is resolving (prevents cross-ticket contamination). On any failure returns a structured rejection to the agent loop naming the failed check; the agent retries within its `maxIterationsPerTask` budget. Passing calls proceed to the simulated ticket store.

## 9. Agent prompts

- `BugResolutionAgent` → `prompts/bug-resolution-agent.md`. The single decision-making LLM. System prompt instructs it to gather context via `search_web` and `read_ticket` tool calls, then propose a resolution via a single `write_ticket` tool call that must satisfy the guardrail.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Java NullPointerException seed; within 30 s the resolution appears with `FIXED` status, a non-empty `resolutionBody`, and a tool-call trace showing at least one `search_web` and one `write_ticket` call.
2. **J2** — The agent's first `write_ticket` call has an empty `resolutionBody` — the guardrail blocks it; the agent corrects and retries; the UI eventually shows a valid resolution. The rejected call appears in the tool-call trace with a `REJECTED` badge.
3. **J3** — A bug submitted with `priority = CRITICAL` and all `search_web` results returning empty snippets still resolves; `confidenceLevel` is `LOW` and the `resolutionBody` says the agent found no matching context.
4. **J4** — The SSE stream delivers `SUBMITTED`, `ENRICHED`, `INVESTIGATING`, `RESOLVED` events in order; a browser that connects after `RESOLVED` receives the full row on the first event.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named bug-assistant demonstrating the single-agent × dev-code cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-bug-assistant. Java package io.akka.samples.softwarebugassistantpython.
Akka 3.6.0. HTTP port 9702.

Components to wire (exactly):

- 1 AutonomousAgent BugResolutionAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/bug-resolution-agent.md>) and
  .capability(TaskAcceptance.of(RESOLVE_BUG).maxIterationsPerTask(5)). The agent is
  given three tools: search_web(query: String) → List<SearchResult>,
  read_ticket(ticketId: String) → TicketMetadata, and write_ticket(ticketId: String,
  status: String, resolutionBody: String, confidenceLevel: String) → String. The agent
  is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml) bound
  to the write_ticket tool, registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries within its 5-iteration budget.

- 1 Workflow BugWorkflow per bugId with three steps:
  * awaitEnrichedStep — polls BugEntity.getBug every 1s; on bug.ticket().isPresent()
    advances to investigateStep. WorkflowSettings.stepTimeout 15s.
  * investigateStep — emits InvestigationStarted, then calls componentClient
    .forAutonomousAgent(BugResolutionAgent.class, "resolver-" + bugId)
    .runSingleTask(TaskDef.instructions(formatBugDetails(bug))) — returns a taskId,
    then forTask(taskId).result(RESOLVE_BUG) to fetch the Resolution. On success calls
    BugEntity.recordResolution(resolution). WorkflowSettings.stepTimeout 90s with
    defaultStepRecovery maxRetries(2).failoverTo(BugWorkflow::error).
  * closeStep — calls BugEntity.close() to mark the bug resolved; transitions entity
    to RESOLVED. WorkflowSettings.stepTimeout 5s. error step transitions entity to
    FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BugEntity (one per bugId). State Bug{bugId: String,
  report: Optional<BugReport>, ticket: Optional<TicketMetadata>,
  resolution: Optional<Resolution>, status: BugStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. BugStatus enum: SUBMITTED, ENRICHED, INVESTIGATING,
  RESOLVED, FAILED. Events: BugSubmitted{report}, TicketEnriched{ticket},
  InvestigationStarted{}, ResolutionRecorded{resolution}, BugFailed{reason}. Commands:
  submit, attachEnriched, markInvestigating, recordResolution, close, fail, getBug.
  emptyState() returns Bug.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer TicketSyncConsumer subscribed to BugEntity events; on BugSubmitted runs
  a simulated ticket-fetch that builds TicketMetadata (minting a PROJ-NNNN key
  deterministically from the bugId, assigning a random team member from a seeded list,
  attaching the priority as a label), then calls BugEntity.attachEnriched(ticket).
  After attachEnriched lands, the same Consumer starts a BugWorkflow with id =
  "bug-" + bugId.

- 1 View BugView with row type BugRow (mirrors Bug minus any audit-only raw fields).
  Table updater consumes BugEntity events. ONE query getAllBugs: SELECT * AS bugs FROM
  bug_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * BugEndpoint at /api with POST /bugs (body {title, description, priority, component,
    reportedBy}; mints bugId; calls BugEntity.submit; returns {bugId}), GET /bugs
    (list from getAllBugs, sorted newest-first), GET /bugs/{id} (one row), GET
    /bugs/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- BugTasks.java declaring one Task<R> constant: RESOLVE_BUG = Task.name("Resolve bug")
  .description("Investigate the bug using available tools and write a resolution to
  the ticket").resultConformsTo(Resolution.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records BugReport, Priority, TicketMetadata, SearchResult, Resolution,
  ResolutionStatus, ConfidenceLevel, Bug, BugStatus.

- TicketWriteGuardrail.java implementing the before-tool-call hook bound to the
  write_ticket tool. Reads ticketId, status, resolutionBody, confidenceLevel from the
  tool arguments. Runs three checks listed in eval-matrix.yaml G1 and either passes
  the call through or returns Guardrail.reject(<structured-error>) to force the agent
  loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9702
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-bugs.jsonl with 3 seeded bug reports:
  a Java NullPointerException in an HTTP handler, a Go channel deadlock in a message
  processor, and a Python async race condition in a coroutine chain.

- src/main/resources/sample-events/mock-search-results.jsonl with canned
  search-result payloads matched by keyword so the mock tool handler can return
  realistic snippets for each seed bug.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = automated (the agent writes the ticket directly, no
  human approval gate), oversight.human_in_loop = false (human reviews the ticket
  board at will but is not in the loop per resolution), failure.failure_modes including
  "wrong-ticket-write", "hallucinated-fix", "low-confidence-resolution-accepted",
  "guardrail-budget-exhausted"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/bug-resolution-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: BugAssistant", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = live list of bug cards; right = selected-bug detail with report
  description, ticket metadata, tool-call trace, resolution body, confidence chip).
  Browser title exactly: <title>Akka Sample: BugAssistant</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(bugId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    resolve-bug.json — 6 Resolution entries covering all ResolutionStatus values plus
      all ConfidenceLevel values. Each entry has a non-empty resolutionBody (at least
      2 sentences) and an evidence list with 1–3 SearchResult items. Plus 2
      deliberately MALFORMED write_ticket calls (one with empty resolutionBody; one
      with an unknown status value) — the guardrail blocks both, exercising the retry
      path. The mock selects a malformed entry on the FIRST call for every 3rd bug
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(bugId) helper makes per-bug selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BugResolutionAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion BugTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (investigateStep 90s, awaitEnrichedStep 15s, closeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Bug row record is Optional<T>.
- Lesson 7: BugTasks.java with RESOLVE_BUG = Task.name(...).description(...)
  .resultConformsTo(Resolution.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9702 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (BugResolutionAgent). No LLM call happens outside this agent.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism bound to the write_ticket tool, not as an external post-hoc check.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
