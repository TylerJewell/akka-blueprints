# SPEC — document-forge-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Document Forge (Claude Skills Demo).
**One-line pitch:** A user submits a text prompt and an optional style template; one AI agent calls Claude skills + tools to generate a structured document, validates the output path before writing, and returns a typed `ForgeResult` with the document text and an audit trail entry.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One `DocumentForgeAgent` (AutonomousAgent) carries the entire generation decision; the surrounding components prepare its input and audit its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** intercepts every invocation of the `write_document` tool before it executes — so a bad path or disallowed content category never reaches the in-memory file store.

The blueprint shows that a document-generation agent is not ungoverned just because it is writing its own output: the guardrail sits between the agent's intention and the side-effect, giving the workflow a chance to reject, correct, or log every write.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **document type** from a dropdown (Business Memo, Technical Brief, Marketing Copy, Project Proposal, or Custom) or types a custom type label.
2. The user enters a **prompt** describing the document to generate.
3. Optionally, the user picks a **style template** from a dropdown (Formal, Casual, Technical, Executive Summary) that constrains the agent's output style.
4. The user clicks **Forge document**. The UI POSTs to `/api/forges` and receives a `forgeId`.
5. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `FORGING` — the workflow has started and the agent is active.
6. Within ~10–30 s, the `forgeStep` completes. The card transitions to `FORGE_COMPLETED`. The generated document appears in the right pane: the full text, the output filename the agent chose, and the agent's brief rationale for its structural decisions.
7. Within ~1 s of `FORGE_COMPLETED`, the `auditStep` finishes. The card shows an **audit score** chip (1–5) plus a one-line assessment of output quality (word count adequacy, structural completeness, and absence of disallowed content patterns).
8. The user can submit another forge request; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ForgeEndpoint` | `HttpEndpoint` | `/api/forges/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ForgeEntity`, `ForgeView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ForgeEntity` | `EventSourcedEntity` | Per-forge lifecycle: submitted → forging → completed → audited. Source of truth. | `ForgeEndpoint`, `ForgeOutputConsumer`, `ForgeWorkflow` | `ForgeView` |
| `ForgeOutputConsumer` | `Consumer` | Subscribes to `ForgeCompleted` events; verifies the output against quality rules; calls `ForgeEntity.attachAudit`. | `ForgeEntity` events | `ForgeEntity` |
| `ForgeWorkflow` | `Workflow` | One workflow per forge. Steps: `forgeStep` → `auditStep`. | started by `ForgeEndpoint` after submit lands | `DocumentForgeAgent`, `ForgeEntity` |
| `DocumentForgeAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the prompt and style template in the task definition; calls the `write_document` tool to commit output; returns `ForgeResult`. | invoked by `ForgeWorkflow` | returns forge result |
| `ForgeView` | `View` | Read model: one row per forge for the UI. | `ForgeEntity` events | `ForgeEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ForgeRequest(
    String forgeId,
    String prompt,
    String documentType,
    String styleTemplate,
    String requestedBy,
    Instant submittedAt
) {}

record DocumentOutput(
    String filename,
    String content,
    String formatHint,       // e.g. "markdown", "plain-text"
    int wordCount
) {}

record AuditEntry(
    int score,               // 1..5
    String assessment,       // one-line
    List<String> flaggedPatterns,
    Instant auditedAt
) {}

record ForgeResult(
    String outputFilename,
    String documentContent,
    String agentRationale,
    Instant forgedAt
) {}

record ForgeRun(
    String forgeId,
    Optional<ForgeRequest> request,
    Optional<DocumentOutput> output,
    Optional<AuditEntry> audit,
    ForgeStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ForgeStatus {
    SUBMITTED, FORGING, FORGE_COMPLETED, AUDITED, FAILED
}

enum DocumentType {
    BUSINESS_MEMO, TECHNICAL_BRIEF, MARKETING_COPY, PROJECT_PROPOSAL, CUSTOM
}

enum StyleTemplate {
    FORMAL, CASUAL, TECHNICAL, EXECUTIVE_SUMMARY
}
```

Events on `ForgeEntity`: `ForgeSubmitted`, `ForgeStarted`, `ForgeCompleted`, `ForgeAudited`, `ForgeFailed`.

Every nullable lifecycle field on the `ForgeRun` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/forges` — body `{ prompt, documentType, styleTemplate, requestedBy }` → `{ forgeId }`.
- `GET /api/forges` — list all forges, newest-first.
- `GET /api/forges/{id}` — one forge run.
- `GET /api/forges/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Document Forge</title>`.

The App UI tab is a two-column layout: a left panel with the forge submission form + live list of forged documents (status pill + audit score chip + document type + age) and a right pane with the selected forge's detail — original prompt, style template, generated document text, and audit score chip with assessment.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every invocation of the `write_document` tool by `DocumentForgeAgent`. Asserts the target `filename` does not contain path-traversal sequences (`../`, absolute paths, or disallowed extensions), the `content` field is non-empty and within the configured size limit (128 KB), and the `formatHint` is in the allowed set (`markdown`, `plain-text`, `html`). On any failure, returns a structured `invalid-tool-call` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `DocumentForgeAgent` → `prompts/document-forge.md`. The single decision-making LLM. System prompt instructs it to read the forge request, produce a well-structured document matching the requested type and style, and commit the output via a single `write_document` tool call with a safe relative filename.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a Business Memo prompt with the Formal template; within 30 s the generated document appears in the UI with a non-empty output, a valid filename, and an audit score chip.
2. **J2** — The agent attempts to write to `../../etc/passwd` (mock LLM path) — the `before-tool-call` guardrail rejects it; the agent retries with a safe relative path; the document lands successfully.
3. **J3** — An output document with fewer than 50 words receives an audit score of 1 with a clear assessment; the UI flags the card.
4. **J4** — A forge prompt containing a path-traversal string is submitted; the guardrail log shows one `guardrail.reject` line naming `path-traversal-detected`; no write ever lands with the disallowed path.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named document-forge-agent demonstrating the single-agent × content-editorial
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-content-editorial-document-forge-agent. Java package
io.akka.samples.documentforgeclaudeskillsdemo. Akka 3.6.0. HTTP port 9594.

Components to wire (exactly):

- 1 AutonomousAgent DocumentForgeAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/document-forge.md>) and
  .capability(TaskAcceptance.of(FORGE_DOCUMENT).maxIterationsPerTask(3)).
  The task receives the prompt and style template as its instruction text.
  Output: ForgeResult{outputFilename: String, documentContent: String,
  agentRationale: String, forgedAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the
  agent loop retries within its 3-iteration budget.
  The agent exposes one tool: write_document(filename: String, content: String,
  formatHint: String) — this is the tool the guardrail intercepts.

- 1 Workflow ForgeWorkflow per forgeId with two steps:
  * forgeStep — emits ForgeStarted, then calls componentClient.forAutonomousAgent(
    DocumentForgeAgent.class, "forger-" + forgeId).runSingleTask(
      TaskDef.instructions(formatForgeRequest(run.request))
    ) — returns a taskId, then forTask(taskId).result(FORGE_DOCUMENT) to fetch the result.
    On success calls ForgeEntity.completeForge(forgeResult). WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(ForgeWorkflow::error).
  * auditStep — runs a deterministic rule-based ForgeAuditor (NOT an LLM call) over the
    completed output: checks that documentContent is non-empty and at least 50 words, that
    outputFilename does not contain traversal sequences, that content does not match known
    disallowed patterns (excessive repetition, placeholder text like "TODO" or "LOREM"), and
    that agentRationale is non-empty. Emits ForgeAudited{score: 1-5, assessment: String}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ForgeEntity (one per forgeId). State ForgeRun{forgeId: String,
  request: Optional<ForgeRequest>, output: Optional<DocumentOutput>,
  audit: Optional<AuditEntry>, status: ForgeStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. ForgeStatus enum: SUBMITTED, FORGING,
  FORGE_COMPLETED, AUDITED, FAILED. Events: ForgeSubmitted{request},
  ForgeStarted{}, ForgeCompleted{output}, ForgeAudited{audit}, ForgeFailed{reason}.
  Commands: submit, markForging, completeForge, attachAudit, fail, getForge.
  emptyState() returns ForgeRun.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer ForgeOutputConsumer subscribed to ForgeEntity events; on ForgeCompleted runs
  a quality-check pipeline over the output document (word count, placeholder detection,
  structural completeness) and builds an AuditEntry, then calls
  ForgeEntity.attachAudit(audit). The Consumer starts no workflow — ForgeWorkflow is
  started by ForgeEndpoint immediately after the submit command lands.

- 1 View ForgeView with row type ForgeRow (mirrors ForgeRun). Table updater consumes
  ForgeEntity events. ONE query getAllForges: SELECT * AS forges FROM forge_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ForgeEndpoint at /api with POST /forges (body {prompt, documentType, styleTemplate,
    requestedBy}; mints forgeId; calls ForgeEntity.submit; starts ForgeWorkflow; returns
    {forgeId}), GET /forges (list from getAllForges, sorted newest-first), GET /forges/{id}
    (one row), GET /forges/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ForgeTasks.java declaring one Task<R> constant: FORGE_DOCUMENT = Task.name("Forge
  document").description("Generate a structured document from the provided prompt and style
  template, then write it via the write_document tool").resultConformsTo(ForgeResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ForgeRequest, DocumentOutput, AuditEntry, ForgeResult, ForgeRun,
  ForgeStatus, DocumentType, StyleTemplate.

- WriteDocumentGuardrail.java implementing the before-tool-call hook. Intercepts invocations
  of the write_document tool. Reads the filename, content, and formatHint arguments from
  the tool call. Checks: (1) filename must not contain "../", must not start with "/", and
  must have an allowed extension (.md, .txt, .html); (2) content must be non-empty and at
  most 131072 bytes; (3) formatHint must be one of {markdown, plain-text, html}. On any
  failure returns Guardrail.reject(<structured-error>) naming the failed check. Passing
  tool calls flow through to the agent's tool handler.

- ForgeAuditor.java — pure deterministic logic (no LLM). Inputs: ForgeResult. Outputs:
  AuditEntry. Scoring rubric: +1 if wordCount >= 50; +1 if no placeholder tokens ("TODO",
  "LOREM", "PLACEHOLDER", "INSERT HERE"); +1 if outputFilename has an allowed extension;
  +1 if agentRationale is non-empty and at least 10 words; +1 if content has at least 2
  distinct paragraph breaks. Score out of 5. Documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9594 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The DocumentForgeAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/forge-templates.jsonl with 4 seeded forge requests:
  a formal business memo (150-word prompt), a technical API reference section (200-word
  prompt), a marketing campaign brief (120-word prompt), and a project proposal executive
  summary (180-word prompt).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (forge prompts
  are editorial by default; deployer marks pii true if prompts may contain personal data),
  decisions.authority_level = autonomous (the agent produces the final document without
  human review in the default flow), oversight.human_in_loop = false (output is advisory
  and content-only; a deployer adding human review should flip this), failure.failure_modes
  including "path-traversal-attempt", "disallowed-content-generation",
  "placeholder-polluted-output", "oversized-output"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/document-forge.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Document Forge (Claude Skills Demo)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  forge submission form + live list of forge cards; right = selected-forge detail with
  original prompt, style template, generated document text, and audit score chip).
  Browser title exactly: <title>Akka Sample: Document Forge</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    forge-document.json — 6 ForgeResult entries covering all four document types.
      Each entry has a non-empty documentContent (at least 3 paragraphs), a safe relative
      filename (e.g. "business-memo-2026.md"), a non-empty agentRationale, and a realistic
      forgedAt. Plus 2 deliberately MALFORMED entries — one where filename contains "../",
      one where content is fewer than 10 words — to exercise the guardrail retry path. The
      mock selects a malformed entry on the first iteration of every 3rd forge (modulo seed)
      so J2 is reproducible.
- A MockModelProvider.seedFor(forgeId) helper makes per-forge selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DocumentForgeAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ForgeTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (forgeStep
  60s, auditStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ForgeRun record is Optional<T>.
- Lesson 7: ForgeTasks.java with FORGE_DOCUMENT = Task.name(...).description(...)
  .resultConformsTo(ForgeResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9594 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (DocumentForgeAgent).
  The audit scorer (ForgeAuditor.java) is rule-based and does NOT make an LLM call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the tool call completes.
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
