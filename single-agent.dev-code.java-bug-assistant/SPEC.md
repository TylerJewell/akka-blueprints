# SPEC — java-bug-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SoftwareBugAssistant.
**One-line pitch:** A user submits a bug report; one AI agent reads the normalized report (passed as a task attachment, never as inline prompt text), searches related tickets and known fixes, and returns a structured resolution recommendation — RESOLVED / NEEDS_INVESTIGATION / ESCALATE — with a ranked list of candidate fixes.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `BugResolutionAgent` (AutonomousAgent) carries the entire diagnosis; surrounding components prepare its input and audit its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** intercepts every agent tool call before it is executed. When the agent attempts to write to the ticket system (create or update a ticket), the guardrail validates that the recommendation is well-formed — correct status enum, non-empty candidate-fix list, a non-empty rationale — before allowing the write to proceed. Ill-formed writes are blocked and the agent is instructed to revise its plan on the next iteration.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — the guardrail sits between the agent's intent and the ticket system, preventing partial or malformed data from entering the authoritative store.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a bug report into the **Bug report** textarea (or picks one of three seeded examples — a null-pointer exception trace, an HTTP 502 intermittent failure report, and a database connection pool exhaustion).
2. The user picks a **Bug category** from a dropdown (`runtime-error`, `network-failure`, `resource-exhaustion`) or leaves it as `auto-detect`.
3. The user clicks **Submit bug**. The UI POSTs to `/api/bugs` and receives a `bugId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `NORMALIZED` — the cleaned report is visible in the card detail, with a small list of redaction categories the normalizer applied.
5. Within ~2 s, it transitions to `SEARCHING` and then `SEARCH_COMPLETE` — the related-ticket panel shows up to 5 matched tickets with their IDs, summaries, and resolution statuses.
6. Within ~10–30 s, the workflow's `resolveStep` completes. The card transitions to `RESOLVING` then `RECOMMENDATION_RECORDED`. The recommendation appears: a top-level action badge (RESOLVED / NEEDS_INVESTIGATION / ESCALATE), a short rationale paragraph, and a ranked candidate-fix list (fix id, confidence score, description, source ticket or knowledge-base entry).
7. Within ~1 s of the recommendation, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the recommendation's evidence is solid.
8. The user can submit another bug; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BugEndpoint` | `HttpEndpoint` | `/api/bugs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BugEntity`, `BugView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BugEntity` | `EventSourcedEntity` | Per-bug lifecycle: submitted → normalized → searching → search-complete → resolving → recommendation → eval. Source of truth. | `BugEndpoint`, `BugNormalizer`, `TicketSearcher`, `ResolutionWorkflow` | `BugView` |
| `BugNormalizer` | `Consumer` | Subscribes to `BugSubmitted` events; removes internal hostnames, stack-trace tokens, and credentials; emits `BugNormalized`. | `BugEntity` events | `BugEntity` |
| `TicketSearcher` | `Consumer` | Subscribes to `BugNormalized` events; queries in-process ticket store for related open and resolved tickets; emits `SearchResultsAttached`. | `BugEntity` events | `BugEntity` |
| `ResolutionWorkflow` | `Workflow` | One workflow per bug. Steps: `awaitNormalizedStep` → `searchStep` → `resolveStep` → `evalStep`. | started by `TicketSearcher` once search lands | `BugResolutionAgent`, `BugEntity` |
| `BugResolutionAgent` | `AutonomousAgent` | The one decision-making LLM. Receives search results as task instructions and the normalized bug report as a task attachment; returns `ResolutionRecommendation`. | invoked by `ResolutionWorkflow` | returns recommendation |
| `BugView` | `View` | Read model: one row per bug for the UI. | `BugEntity` events | `BugEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CandidateFix(
    String fixId,
    int confidenceScore,      // 0..100
    String description,
    String sourceRef          // ticket id or KB entry id
) {}

record SearchResult(
    String ticketId,
    String summary,
    TicketStatus ticketStatus,
    String resolution         // may be empty for open tickets
) {}
enum TicketStatus { OPEN, IN_PROGRESS, RESOLVED, CLOSED, WONT_FIX }

record BugReport(
    String bugId,
    String title,
    String rawReport,
    BugCategory category,
    String submittedBy,
    Instant submittedAt
) {}
enum BugCategory { RUNTIME_ERROR, NETWORK_FAILURE, RESOURCE_EXHAUSTION, UNKNOWN }

record NormalizedReport(
    String cleanedReport,
    List<String> redactionCategories  // e.g. ["internal-hostname","stack-trace-token","credential"]
) {}

record ResolutionRecommendation(
    ResolutionAction action,
    String rationale,
    List<CandidateFix> candidateFixes,
    Instant decidedAt
) {}
enum ResolutionAction { RESOLVED, NEEDS_INVESTIGATION, ESCALATE }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Bug(
    String bugId,
    Optional<BugReport> report,
    Optional<NormalizedReport> normalized,
    Optional<List<SearchResult>> searchResults,
    Optional<ResolutionRecommendation> recommendation,
    Optional<EvalResult> eval,
    BugStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BugStatus {
    SUBMITTED, NORMALIZED, SEARCHING, SEARCH_COMPLETE, RESOLVING,
    RECOMMENDATION_RECORDED, EVALUATED, FAILED
}
```

Events on `BugEntity`: `BugSubmitted`, `BugNormalized`, `SearchStarted`, `SearchResultsAttached`, `ResolutionStarted`, `RecommendationRecorded`, `EvaluationScored`, `BugFailed`.

Every nullable lifecycle field on the `Bug` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/bugs` — body `{ title, rawReport, category, submittedBy }` → `{ bugId }`.
- `GET /api/bugs` — list all bugs, newest-first.
- `GET /api/bugs/{id}` — one bug.
- `GET /api/bugs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Software Bug Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted bugs (status pill + action badge + age) and a right pane with the selected bug's detail — normalized report preview, related tickets panel, recommendation rationale, candidate-fix ranked list, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call the `BugResolutionAgent` attempts. When the agent calls `createTicket` or `updateTicket`, the guardrail validates that the candidate `ResolutionRecommendation` is well-formed before allowing the write: (1) `action` is one of `RESOLVED`, `NEEDS_INVESTIGATION`, `ESCALATE`; (2) `candidateFixes` is non-empty; (3) `rationale` is non-empty and at least 20 characters; (4) every `CandidateFix.confidenceScore` is in `0..100`. On failure, blocks the tool call and returns a structured rejection to the agent loop so the agent revises its plan within its iteration budget.

## 9. Agent prompts

- `BugResolutionAgent` → `prompts/bug-resolution-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached normalized bug report, review the provided search results, and return one ranked list of candidate fixes with a top-level resolution action.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the null-pointer seed; within 30 s the recommendation appears with a ranked candidate-fix list and an eval score chip.
2. **J2** — The agent attempts a ticket write with a malformed recommendation on a review (mock LLM path) — the `before-tool-call` guardrail blocks it; the agent revises; a well-formed recommendation lands; the UI never displays the blocked payload.
3. **J3** — A recommendation with all candidate fixes at `confidenceScore == 0` receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A bug report containing `DB_PASSWORD=s3cr3t`, an internal hostname (`db-prod-01.internal`), and a JWT token is submitted; the LLM call log shows only the redacted forms; the entity's `report.rawReport` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named softwarebugassistantjava demonstrating the single-agent × dev-code cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-java-bug-assistant. Java package io.akka.samples.softwarebugassistantjava.
Akka 3.6.0. HTTP port 9739.

Components to wire (exactly):

- 1 AutonomousAgent BugResolutionAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/bug-resolution-agent.md>) and
  .capability(TaskAcceptance.of(BugTasks.RESOLVE_BUG).maxIterationsPerTask(4)). The task
  receives search results as its instruction text and the normalized bug report as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is
  the canonical call). Output: ResolutionRecommendation{action: ResolutionAction
  (RESOLVED/NEEDS_INVESTIGATION/ESCALATE), rationale: String, candidateFixes:
  List<CandidateFix>, decidedAt: Instant}. The agent is configured with a before-tool-call
  guardrail (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration
  block. On guardrail rejection the agent loop retries within its 4-iteration budget.

- 1 Workflow ResolutionWorkflow per bugId with four steps:
  * awaitNormalizedStep — polls BugEntity.getBug every 1s; on bug.normalized().isPresent()
    advances to searchStep. WorkflowSettings.stepTimeout 15s.
  * searchStep — emits SearchStarted, reads bug.normalized, then calls
    componentClient.forEventSourcedEntity(BugEntity.class, bugId)
    .call(BugEntity::attachSearchResults) with the TicketSearcher's results already on the
    entity (the workflow checks bug.searchResults().isPresent() after a 2s pause). Advances
    to resolveStep. WorkflowSettings.stepTimeout 30s.
  * resolveStep — emits ResolutionStarted, then calls componentClient.forAutonomousAgent(
    BugResolutionAgent.class, "resolver-" + bugId).runSingleTask(
      TaskDef.instructions(formatSearchResults(bug.searchResults))
        .attachment("bug-report.txt", bug.normalized.cleanedReport.getBytes())
    ) — returns a taskId, then forTask(taskId).result(BugTasks.RESOLVE_BUG) to fetch the
    recommendation. On success calls BugEntity.recordRecommendation(recommendation).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ResolutionWorkflow::error).
  * evalStep — runs a deterministic rule-based RecommendationScorer (NOT an LLM call) over
    the recorded recommendation: checks that every CandidateFix has a non-empty description
    and sourceRef, that confidenceScores are not all zero, that rationale is actionable, and
    that the action is appropriate for the top fix's confidence score (a RESOLVED action with
    all fixes below confidence 30 loses 1 point). Emits EvaluationScored{score: 1-5,
    rationale: String}. WorkflowSettings.stepTimeout 5s. error step transitions the entity
    to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BugEntity (one per bugId). State Bug{bugId: String, report:
  Optional<BugReport>, normalized: Optional<NormalizedReport>, searchResults:
  Optional<List<SearchResult>>, recommendation: Optional<ResolutionRecommendation>,
  eval: Optional<EvalResult>, status: BugStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. BugStatus enum: SUBMITTED, NORMALIZED, SEARCHING,
  SEARCH_COMPLETE, RESOLVING, RECOMMENDATION_RECORDED, EVALUATED, FAILED. Events:
  BugSubmitted{report}, BugNormalized{normalized}, SearchStarted{}, SearchResultsAttached{
  searchResults}, ResolutionStarted{}, RecommendationRecorded{recommendation},
  EvaluationScored{eval}, BugFailed{reason}. Commands: submit, attachNormalized,
  startSearch, attachSearchResults, startResolving, recordRecommendation, recordEvaluation,
  fail, getBug. emptyState() returns Bug.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer BugNormalizer subscribed to BugEntity events; on BugSubmitted runs a
  regex+heuristic normalization pipeline (internal hostnames, stack-trace tokens that look
  like internal addresses, credential patterns like KEY=value or password= patterns, JWT-like
  base64 tokens) over rawReport, computes the list of redaction categories found, builds
  NormalizedReport, then calls BugEntity.attachNormalized(normalized).

- 1 Consumer TicketSearcher subscribed to BugEntity events; on BugNormalized calls the
  in-process TicketStore (a singleton populated from
  src/main/resources/sample-events/seeded-tickets.jsonl at startup) to find up to 5 tickets
  whose summary overlaps with the normalized report's key terms, builds List<SearchResult>,
  then calls BugEntity.attachSearchResults(results).

- 1 View BugView with row type BugRow (mirrors Bug minus report.rawReport — the audit log
  keeps the raw; the view holds the normalized form for the UI). Table updater consumes
  BugEntity events. ONE query getAllBugs: SELECT * AS bugs FROM bug_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * BugEndpoint at /api with POST /bugs (body {title, rawReport, category, submittedBy};
    mints bugId; calls BugEntity.submit; returns {bugId}), GET /bugs (list from getAllBugs,
    sorted newest-first), GET /bugs/{id} (one row), GET /bugs/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- BugTasks.java declaring one Task<R> constant: RESOLVE_BUG = Task.name("Resolve bug")
  .description("Read the attached normalized bug report and search results, then produce a
  ResolutionRecommendation with ranked candidate fixes").resultConformsTo(
  ResolutionRecommendation.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records CandidateFix, SearchResult, TicketStatus, BugReport, BugCategory,
  NormalizedReport, ResolutionRecommendation, ResolutionAction, EvalResult, Bug, BugStatus.

- TicketWriteGuardrail.java implementing the before-tool-call hook. When the agent calls
  createTicket or updateTicket, reads the candidate ResolutionRecommendation from the call
  arguments, runs the four checks listed in eval-matrix.yaml G1, and either passes the tool
  call through or returns Guardrail.reject(<structured-error>) to force the agent loop to
  revise its plan.

- RecommendationScorer.java — pure deterministic logic (no LLM). Inputs:
  ResolutionRecommendation and the List<SearchResult>. Outputs: EvalResult. Scoring rubric
  documented in Javadoc on the class.

- TicketStore.java — an in-process singleton loaded from seeded-tickets.jsonl. Exposes
  List<SearchResult> search(String normalizedText) for TicketSearcher to call. No external
  HTTP; no database; runs entirely in memory.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9739 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/bug-taxonomy.jsonl with 3 seeded bug categories and
  example reports: a null-pointer exception in a payment service, an HTTP 502 intermittent
  failure in an API gateway, a database connection pool exhaustion.

- src/main/resources/sample-events/seeded-tickets.jsonl with 10 pre-seeded tickets covering
  all three bug categories, in statuses OPEN, RESOLVED, and CLOSED, with resolution fields
  for the closed ones. Used by TicketStore at startup.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with decisions.authority_level = recommend-only,
  oversight.human_in_loop = true (a developer reads the recommendation before acting),
  failure.failure_modes including "hallucinated-fix", "missed-related-ticket",
  "overconfident-resolution", "partial-write-to-ticket-system"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/bug-resolution-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Software Bug Assistant (Java)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of bug cards; right = selected-bug detail with normalized report preview, related
  tickets panel, recommendation rationale, candidate-fix ranked list, and eval-score chip).
  Browser title exactly: <title>Akka Sample: Software Bug Assistant</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(bugId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    resolve-bug.json — 8 ResolutionRecommendation entries covering all three ResolutionAction
      values. Each entry has a rationale paragraph and a candidateFixes list of 2–4 entries,
      each with a non-empty description, a non-empty sourceRef (a seeded ticket id or a
      knowledge-base entry id), and a confidenceScore between 30 and 95. Actions vary
      realistically across entries. Plus 2 deliberately MALFORMED entries (one with an empty
      candidateFixes list; one with a confidenceScore outside 0..100) — the guardrail blocks
      both, exercising the retry path. The mock selects a malformed entry on the FIRST
      iteration of every 3rd bug (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(bugId) helper makes per-bug selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BugResolutionAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BugTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (resolveStep
  60s, awaitNormalizedStep 15s, searchStep 30s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Bug row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: BugTasks.java with RESOLVE_BUG = Task.name(...).description(...)
  .resultConformsTo(ResolutionRecommendation.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9739 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (BugResolutionAgent).
  The on-decision eval is rule-based (RecommendationScorer.java) and does NOT make an LLM
  call — keeping the pattern's "one agent" promise honest.
- The bug report is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated resolveStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns. Lesson 1's AutonomousAgent
  contract is the authoritative reference for how the hook is registered.
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
