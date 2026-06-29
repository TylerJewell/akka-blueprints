# SPEC — http-content-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Claude AI Content Generator.
**One-line pitch:** A caller POSTs a content brief; one AI agent produces a finished blog post or
social copy, a `before-agent-response` guardrail validates the draft for brand tone and safety
before it is returned, and every generated piece is stored on an event-sourced entity for audit.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One
`ContentGeneratorAgent` (AutonomousAgent) carries the entire content-creation decision; the
surrounding components only deliver its brief, audit its draft, and persist the result. One
governance mechanism is wired around the agent:

- A **before-agent-response guardrail** (`DraftGuardrail`) validates the agent's draft on every
  turn: well-formed `GeneratedContent` JSON, required fields present (`title`, `body`, `format`),
  body word count within declared limits, no prohibited-word matches from the configurable brand
  ruleset. A failing draft triggers a retry inside the same task.

The blueprint shows that a single content-generation agent does not need to be ungoverned: even a
one-step generation pipeline benefits from a structured exit check before the output reaches
callers.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in the **Content brief** form: a `topic` field, a `targetAudience` field, an
   `outputFormat` dropdown (BLOG_POST / SOCIAL_POST / EMAIL_TEASER / PRODUCT_DESCRIPTION), and an
   optional `wordCountHint` (50–2000). Or the user picks one of three seeded briefs — a
   technology-trends blog post, a social media campaign post, and an email teaser for a product
   launch.
2. The user clicks **Generate**. The UI POSTs to `/api/jobs` and receives a `jobId`.
3. The card appears in the live list in `REQUESTED` state.
4. Within ~10–30 s the workflow's `generateStep` completes. The card transitions through
   `GENERATING` to `APPROVED`. The finished piece appears: a `title`, a `body` text block, an
   `outputFormat` badge, and a `wordCount`.
5. If the agent's first draft fails the guardrail, the card stays in `GENERATING` while the retry
   completes — the caller never sees the rejected draft.
6. The user can submit another brief; the live list keeps the full history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ContentEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ContentJobEntity`, `ContentView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ContentJobEntity` | `EventSourcedEntity` | Per-job lifecycle: requested → generating → approved / failed. Source of truth. | `ContentEndpoint`, `ContentWorkflow` | `ContentView` |
| `ContentWorkflow` | `Workflow` | One workflow per job. Steps: `generateStep` → (on failure) `errorStep`. | started by `ContentEndpoint` after entity creation | `ContentGeneratorAgent`, `ContentJobEntity` |
| `ContentGeneratorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the content brief as task instructions; returns `GeneratedContent`. | invoked by `ContentWorkflow` | returns draft |
| `DraftGuardrail` | supporting class | `before-agent-response` hook registered on `ContentGeneratorAgent`. Validates the draft; rejects on any failed check; passes through otherwise. | wired to `ContentGeneratorAgent` | returns decision to agent loop |
| `ContentView` | `View` | Read model: one row per job for the UI. | `ContentJobEntity` events | `ContentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum OutputFormat { BLOG_POST, SOCIAL_POST, EMAIL_TEASER, PRODUCT_DESCRIPTION }

record ContentBrief(
    String topic,
    String targetAudience,
    OutputFormat outputFormat,
    int wordCountHint,       // 50–2000; default 300 when not supplied
    String submittedBy,
    Instant submittedAt
) {}

record GeneratedContent(
    String title,
    String body,
    OutputFormat format,
    int wordCount,
    Instant generatedAt
) {}

record GuardrailResult(
    boolean passed,
    List<String> failedChecks   // empty when passed
) {}

record ContentJob(
    String jobId,
    Optional<ContentBrief> brief,
    Optional<GeneratedContent> content,
    JobStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus {
    REQUESTED, GENERATING, APPROVED, FAILED
}
```

Events on `ContentJobEntity`: `JobRequested`, `GenerationStarted`, `ContentApproved`, `JobFailed`.

Every nullable lifecycle field on the `ContentJob` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/jobs` — body `{ topic, targetAudience, outputFormat, wordCountHint, submittedBy }` → `{ jobId }`.
- `GET /api/jobs` — list all jobs, newest-first.
- `GET /api/jobs/{id}` — one job.
- `GET /api/jobs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser
title: `<title>Akka Sample: Claude AI Content Generator</title>`.

The App UI tab is a two-column layout: a left rail with the content-brief submission form above a
live list of submitted jobs (status pill + format badge + age) and a right pane with the selected
job's detail — brief topic, target audience, format, word-count hint, the generated title and body
block, and a word-count badge.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index
(Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and
`themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black
and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`DraftGuardrail`): runs on every turn of
  `ContentGeneratorAgent`. Asserts the candidate response is well-formed `GeneratedContent` JSON,
  that `title`, `body`, and `format` are all non-empty, that `wordCount` is within ±50 % of
  `wordCountHint`, and that `body` contains no string from the configurable prohibited-word list
  (loaded from `src/main/resources/brand-rules/prohibited-words.txt` at startup). On failure,
  returns a structured `invalid-draft` error to the agent loop identifying which check(s) failed,
  so the agent can correct the specific issue in its next iteration.

## 9. Agent prompts

- `ContentGeneratorAgent` → `prompts/content-generator.md`. The single decision-making LLM.
  System prompt instructs it to write the requested content type for the named audience, stay
  within the word count, and return the result as a `GeneratedContent` JSON object.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the technology-trends blog brief; within 30 s the approved content
   appears with a non-empty `title`, a `body` of ≥ 100 words, format `BLOG_POST`, and a
   `wordCount`.
2. **J2** — The agent's first draft on a social-post brief contains a word from the prohibited
   list (mock LLM path) — the guardrail rejects it; the second iteration produces a clean draft;
   the UI shows only the approved text.
3. **J3** — A brief whose mock response returns an incomplete `GeneratedContent` (missing `title`)
   is caught by the guardrail; the job retries; the corrected response includes all required
   fields.
4. **J4** — Four briefs submitted in quick succession each produce independent jobs; no content
   from one job appears in another.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole
— Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named http-content-agent demonstrating the single-agent × content-editorial cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-content-editorial-http-content-agent. Java package
io.akka.samples.claudeaicontentgenerator. Akka 3.6.0. HTTP port 9657.

Components to wire (exactly):

- 1 AutonomousAgent ContentGeneratorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/content-generator.md>) and
  .capability(TaskAcceptance.of(GENERATE_CONTENT).maxIterationsPerTask(3)). The task receives
  the full content brief as its instruction text. Output: GeneratedContent{title: String,
  body: String, format: OutputFormat, wordCount: int, generatedAt: Instant}. The agent is
  configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via
  the agent's guardrail-configuration block. On guardrail rejection the agent loop retries the
  draft within its 3-iteration budget.

- 1 Workflow ContentWorkflow per jobId with two steps:
  * generateStep — emits GenerationStarted, then calls componentClient.forAutonomousAgent(
    ContentGeneratorAgent.class, "generator-" + jobId).runSingleTask(
      TaskDef.instructions(formatBrief(job.brief))
    ) — returns a taskId, then forTask(taskId).result(GENERATE_CONTENT) to fetch the draft.
    On success calls ContentJobEntity.approveContent(content).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(ContentWorkflow::error).
  * error step — calls ContentJobEntity.failJob(reason). WorkflowSettings.stepTimeout 5s.
    Transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ContentJobEntity (one per jobId). State ContentJob{jobId: String,
  brief: Optional<ContentBrief>, content: Optional<GeneratedContent>, status: JobStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. JobStatus enum: REQUESTED, GENERATING,
  APPROVED, FAILED. Events: JobRequested{brief}, GenerationStarted{}, ContentApproved{content},
  JobFailed{reason}. Commands: requestJob, startGeneration, approveContent, failJob, getJob.
  emptyState() returns ContentJob.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View ContentView with row type ContentRow (mirrors ContentJob). Table updater consumes
  ContentJobEntity events. ONE query getAllJobs: SELECT * AS jobs FROM content_job_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ContentEndpoint at /api with POST /jobs (body {topic, targetAudience, outputFormat,
    wordCountHint, submittedBy}; mints jobId; calls ContentJobEntity.requestJob; starts
    ContentWorkflow; returns {jobId}), GET /jobs (list from getAllJobs, sorted newest-first),
    GET /jobs/{id} (one row), GET /jobs/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ContentTasks.java declaring one Task<R> constant: GENERATE_CONTENT = Task
  .name("Generate content").description("Write content matching the supplied brief and return
  it as a GeneratedContent record").resultConformsTo(GeneratedContent.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records ContentBrief, GeneratedContent, GuardrailResult, ContentJob, OutputFormat,
  JobStatus.

- DraftGuardrail.java implementing the before-agent-response hook. Reads the candidate
  GeneratedContent from the LLM response, runs the four checks listed in eval-matrix.yaml G1
  (well-formed JSON, non-empty title/body/format, word-count within tolerance, no prohibited
  words), and either passes the response through or returns Guardrail.reject(<structured-error>)
  naming the specific failed check(s) to guide the agent's retry.

- src/main/resources/brand-rules/prohibited-words.txt — one word or phrase per line; loaded
  at startup by DraftGuardrail. Seeded with 10 generic brand-safety terms (e.g.,
  "guaranteed results", "risk-free", "100% proven").

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9657 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ContentGeneratorAgent.definition()
  binds the configured provider via the per-agent or default provider pattern from the
  akka-context docs.

- src/main/resources/sample-events/content-briefs.jsonl with 3 seeded briefs:
  a technology-trends blog post (wordCountHint 600), a social media campaign post for a
  product launch (wordCountHint 120), and an email teaser for a new feature release
  (wordCountHint 200).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for content-editorial domain.

- prompts/content-generator.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Claude AI Content Generator",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  brief submission form + live list of job cards; right = selected-job detail with brief
  fields, status pill, format badge, generated title, body block, and word-count badge).
  Browser title exactly: <title>Akka Sample: Claude AI Content Generator</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(jobId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-content.json — 8 GeneratedContent entries covering all four OutputFormat values.
      Each entry has a non-empty title, a body of ≥ 100 words, a realistic wordCount, and
      generatedAt set to a recent timestamp. Plus 2 deliberately MALFORMED entries: one with
      a null title (exercises the missing-field guardrail check); one whose body contains a
      prohibited word from prohibited-words.txt (exercises the prohibited-word check). The
      mock should select a malformed entry on the FIRST iteration of every 3rd job (modulo
      seed) so J2 and J3 are reproducible.
- A MockModelProvider.seedFor(jobId) helper makes per-job selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ContentGeneratorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ContentTasks.java MUST exist.
- Lesson 4: the generateStep has an explicit stepTimeout of 90s; the error step has 5s.
- Lesson 6: every nullable lifecycle field on the ContentJob row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ContentTasks.java with GENERATE_CONTENT = Task.name(...).description(...)
  .resultConformsTo(GeneratedContent.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9657 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ContentGeneratorAgent).
  No second agent is present.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not
stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without
prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept
   defaults; pick the most conservative option if anything is ambiguous, consistent with Sections
   4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure,
   continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a
one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step
takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the
user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml`
written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile
error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
