# SPEC — localwiki-ingest-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** LocalWiki Demo Agent.
**One-line pitch:** A user submits a URL, image, or PDF; one AI agent fetches and parses the source (passed as a task attachment, never as inline prompt text) and files a structured markdown wiki page — title, summary, body, and category tags — in one call.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One `WikiIngestAgent` (AutonomousAgent) carries the entire filing decision; the surrounding components only prepare its input and guard its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw fetched content and the agent call — so the model never sees identifiers scraped from the source.
- A **before-tool-call guardrail** fires on every proposed `write_wiki_page` tool call: validates the target path against a blocklist of forbidden prefixes (e.g., `../`, `/etc/`, `/admin/`), asserts the namespace is one of the declared allowed namespaces, and confirms the proposed page title is non-empty. A rejected tool call surfaces a structured error to the agent so it can propose a corrected path on the next iteration.

The blueprint shows that a single-agent pattern is not ungoverned — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **Source type** (URL / Image / PDF) and enters the source address, or clicks one of three seeded examples.
2. The user selects a **Target namespace** from a dropdown (`/docs`, `/blog`, `/reference`) or types a custom one.
3. The user clicks **Submit for ingest**. The UI POSTs to `/api/ingests` and receives an `ingestId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SANITIZED` — the redacted content preview is visible in the card detail with PII category chips.
5. Within ~10–30 s, the workflow's `ingestStep` completes. The card transitions to `INGESTING` then `PAGE_FILED`. The filed page appears: title, summary, body (markdown), and category tags. The target path shows the namespace and slug the agent chose.
6. The user can submit another source; the live list keeps all history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `IngestEndpoint` | `HttpEndpoint` | `/api/ingests/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `IngestEntity`, `WikiPageView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `IngestEntity` | `EventSourcedEntity` | Per-ingest lifecycle: submitted → sanitized → ingesting → page_filed. Source of truth. | `IngestEndpoint`, `ContentSanitizer`, `IngestWorkflow` | `WikiPageView` |
| `ContentSanitizer` | `Consumer` | Subscribes to `SourceSubmitted` events; fetches source; redacts PII; calls `IngestEntity.attachSanitized`. | `IngestEntity` events | `IngestEntity` |
| `IngestWorkflow` | `Workflow` | One workflow per ingest. Steps: `awaitSanitizedStep` → `ingestStep`. | started by `ContentSanitizer` once sanitized event lands | `WikiIngestAgent`, `IngestEntity` |
| `WikiIngestAgent` | `AutonomousAgent` | The one decision-making LLM. Receives filing instructions in the task definition and the sanitized content as a task attachment; calls `write_wiki_page` tool; returns `WikiPage`. | invoked by `IngestWorkflow` | returns page |
| `WritePageGuardrail` | supporting class | `before-tool-call` guardrail registered on `WikiIngestAgent`. Validates every proposed `write_wiki_page` call. | `WikiIngestAgent` | passes/rejects tool calls |
| `WikiPageView` | `View` | Read model: one row per ingest for the UI. | `IngestEntity` events | `IngestEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SourceSubmission(
    String ingestId,
    SourceType sourceType,
    String sourceAddress,
    String targetNamespace,
    String submittedBy,
    Instant submittedAt
) {}
enum SourceType { URL, IMAGE, PDF }

record SanitizedContent(
    String redactedText,
    String mimeType,
    List<String> piiCategoriesFound
) {}

record WikiPage(
    String title,
    String slug,
    String summary,
    String body,
    List<String> categoryTags,
    String targetPath,
    Instant filedAt
) {}

record Ingest(
    String ingestId,
    Optional<SourceSubmission> submission,
    Optional<SanitizedContent> sanitized,
    Optional<WikiPage> page,
    IngestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    Optional<String> failureReason
) {}

enum IngestStatus {
    SUBMITTED, SANITIZED, INGESTING, PAGE_FILED, FAILED
}
```

Events on `IngestEntity`: `SourceSubmitted`, `ContentSanitized`, `IngestStarted`, `PageFiled`, `IngestFailed`.

Every nullable lifecycle field on the `Ingest` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/ingests` — body `{ sourceType, sourceAddress, targetNamespace, submittedBy }` → `{ ingestId }`.
- `GET /api/ingests` — list all ingests, newest-first.
- `GET /api/ingests/{id}` — one ingest.
- `GET /api/ingests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: LocalWiki Demo Agent</title>`.

The App UI tab is a two-column layout: a left rail with the submission form and the live list of ingests (status pill + source type badge + age) and a right pane with the selected ingest's detail — source address, sanitized content preview, PII category chips, filed page title, summary, body preview, category tags, and target path.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every proposed `write_wiki_page` tool call inside `WikiIngestAgent`. Asserts that the proposed `targetPath` does not begin with any forbidden prefix (`../`, `/etc/`, `/admin/`, `/sys/`), that the `namespace` component of the path is in the declared allowed set (`/docs`, `/blog`, `/reference`, and any deployer-added custom namespace), and that the proposed `title` is non-empty. On failure, returns a structured `tool-call-rejected` error to the agent loop naming which check failed and suggesting the corrected namespace; the agent retries within its iteration budget.
- **S1 — PII sanitizer** (`pii`, applied inside `ContentSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the fetched source before any LLM call. Records which categories were found. The raw fetched content is preserved on the entity for audit; the sanitized form is what the agent receives.

## 9. Agent prompts

- `WikiIngestAgent` → `prompts/wiki-ingest-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached source content, produce a `WikiPage` record with a descriptive title, a one-paragraph summary, a structured markdown body, and appropriate category tags, then call `write_wiki_page` with the page and a target path formed from the user's chosen namespace and a slug derived from the title.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded URL; within 30 s the filed page appears with title, summary, body, category tags, and a target path under the chosen namespace.
2. **J2** — The agent proposes `write_wiki_page` with `targetPath = "../../../etc/passwd"` (mock LLM path) — the `before-tool-call` guardrail rejects it; the agent retries with a path under `/docs/` and the page lands correctly.
3. **J3** — A source containing `bob.smith@example.com` and `SSN 987-65-4321` is submitted; the agent call body shows only `[REDACTED-EMAIL]` and `[REDACTED-SSN]`; the entity's `submission.sourceAddress` retains the raw address for audit.
4. **J4** — A source that times out or returns empty content is recorded as FAILED with reason `"fetch-failed: empty response"`; the entity preserves the partial state; the UI shows the card in red with the failure reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named localwiki-ingest-agent demonstrating the single-agent × content-editorial cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-content-editorial-localwiki-ingest-agent. Java package
io.akka.samples.localwikidemoagent. Akka 3.6.0. HTTP port 9621.

Components to wire (exactly):

- 1 AutonomousAgent WikiIngestAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/wiki-ingest-agent.md>) and
  .capability(TaskAcceptance.of(INGEST_SOURCE).maxIterationsPerTask(3)). The task receives
  filing instructions as its instruction text and the sanitized content as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: WikiPage{title: String, slug: String, summary: String, body: String,
  categoryTags: List<String>, targetPath: String, filedAt: Instant}. The agent is configured with
  a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the tool call
  within its 3-iteration budget.

- 1 Workflow IngestWorkflow per ingestId with two steps:
  * awaitSanitizedStep — polls IngestEntity.getIngest every 1s; on ingest.sanitized().isPresent()
    advances to ingestStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * ingestStep — emits IngestStarted, then calls componentClient.forAutonomousAgent(
    WikiIngestAgent.class, "ingest-agent-" + ingestId).runSingleTask(
      TaskDef.instructions(formatFilingInstructions(ingest.submission.targetNamespace))
        .attachment("source.txt", ingest.sanitized.redactedText.getBytes())
    ) — returns a taskId, then forTask(taskId).result(INGEST_SOURCE) to fetch the filed page.
    On success calls IngestEntity.recordPage(page). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(IngestWorkflow::error).
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity IngestEntity (one per ingestId). State Ingest{ingestId: String,
  submission: Optional<SourceSubmission>, sanitized: Optional<SanitizedContent>,
  page: Optional<WikiPage>, status: IngestStatus, createdAt: Instant,
  finishedAt: Optional<Instant>, failureReason: Optional<String>}. IngestStatus enum:
  SUBMITTED, SANITIZED, INGESTING, PAGE_FILED, FAILED. Events: SourceSubmitted{submission},
  ContentSanitized{sanitized}, IngestStarted{}, PageFiled{page},
  IngestFailed{reason}. Commands: submit, attachSanitized, markIngesting, recordPage, fail,
  getIngest. emptyState() returns Ingest.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer ContentSanitizer subscribed to IngestEntity events; on SourceSubmitted fetches
  the source (URL fetch / image base64 decode / PDF text extract — simulated in-process in
  the sample), then runs a regex+heuristic redaction pipeline (emails, phone numbers,
  SSN-like, payment-card-like, postal addresses, person-name heuristic, account-id-like
  tokens) over the raw text, computes the list of categories found, builds SanitizedContent,
  then calls IngestEntity.attachSanitized(sanitized). After attachSanitized lands, the same
  Consumer starts an IngestWorkflow with id = "ingest-" + ingestId.

- 1 View WikiPageView with row type IngestRow (mirrors Ingest minus submission.sourceAddress
  raw bytes — the audit log keeps the raw; the view holds the sanitized summary for the UI).
  Table updater consumes IngestEntity events. ONE query getAllIngests: SELECT * AS ingests
  FROM wiki_page_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * IngestEndpoint at /api with POST /ingests (body {sourceType, sourceAddress,
    targetNamespace, submittedBy}; mints ingestId; calls IngestEntity.submit; returns
    {ingestId}), GET /ingests (list from getAllIngests, sorted newest-first), GET /ingests/{id}
    (one row), GET /ingests/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- IngestTasks.java declaring one Task<R> constant: INGEST_SOURCE = Task.name("Ingest source")
  .description("Read the attached source content and produce a WikiPage filed to the target namespace")
  .resultConformsTo(WikiPage.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records SourceSubmission, SourceType, SanitizedContent, WikiPage, Ingest, IngestStatus.

- WritePageGuardrail.java implementing the before-tool-call hook. Reads the proposed
  write_wiki_page arguments, runs the three checks listed in eval-matrix.yaml G1
  (path prefix check, namespace allowlist check, non-empty title check), and either passes
  the tool call through or returns Guardrail.reject(<structured-error>) naming which check
  failed and suggesting the corrected namespace.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9621 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The WikiIngestAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-sources.jsonl with 3 seeded sources:
  a public URL pointing to a short technical article (200–400 word simulation), an image
  described as a product screenshot with alt-text and OCR'd captions, and a PDF described
  as a product datasheet. Each contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous
  (the agent writes the page without human approval — deployer may override),
  oversight.human_on_loop = true (a human can review the filed page after the fact),
  failure.failure_modes including "path-traversal-attempt", "namespace-violation",
  "pii-in-wiki-page", "fetch-failed", "malformed-page-structure";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/wiki-ingest-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: LocalWiki Demo Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of ingest cards; right = selected-ingest detail with source
  address, sanitized content preview, PII category chips, filed page title, summary, body
  preview, category tags, and target path).
  Browser title exactly: <title>Akka Sample: LocalWiki Demo Agent</title>. No subtitle on
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(ingestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    ingest-source.json — 6 WikiPage entries covering all three SourceType values with
      realistic titles, slugs, one-paragraph summaries, 4–8 line markdown bodies, and
      2–4 category tags. Target paths are under /docs, /blog, and /reference respectively.
      Plus 2 deliberately MALFORMED entries: one with targetPath = "../../../etc/passwd"
      (triggers the path-prefix check in G1) and one with an empty title (triggers the
      non-empty-title check in G1). The mock selects a malformed entry on the FIRST iteration
      of every 3rd ingest (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(ingestId) helper makes per-ingest selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. WikiIngestAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion IngestTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (ingestStep
  60s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the Ingest row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: IngestTasks.java with INGEST_SOURCE = Task.name(...).description(...)
  .resultConformsTo(WikiPage.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9621 declared explicitly in application.conf's
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
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (WikiIngestAgent).
  WritePageGuardrail is a supporting class registered on the agent — not a second agent.
- The source content is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated ingestStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
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
