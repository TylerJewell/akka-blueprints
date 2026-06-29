# SPEC — ghostwriter

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Ghostwriter.
**One-line pitch:** A user submits a writing brief and a set of the voice owner's past work as attachments; one AI agent reads the samples and drafts new content in the established voice, returning a structured `DraftResult` with tone markers, a ready-to-use draft body, and a fidelity score.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the content-editorial domain. One `GhostwriterAgent` (AutonomousAgent) carries the entire generation decision; the surrounding components prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw brief submission and the agent call — so the model never sees confidential identifiers present in source documents.
- An **after-llm-response guardrail** validates every draft the agent produces: checks that the draft does not reproduce verbatim passages longer than a threshold from the source corpus, that no redaction markers (`[REDACTED-*]`) leak through into the output, that the required content format fields are present, and that the fidelity score is within the declared range. A failing draft triggers a retry inside the same task.

The blueprint shows that a content-generation pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call, addressing the specific risks of voice impersonation and confidential-data leakage that are inherent to this pattern.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a **writing brief** (topic sentence, target length, format: blog-post / memo / release-note / social) and selects a **voice owner** from a dropdown. The voice owner maps to a seeded corpus of 3–5 writing samples.
2. The user may optionally upload an additional writing sample (textarea) to augment the seeded corpus.
3. The user clicks **Generate draft**. The UI POSTs to `/api/drafts` and receives a `draftId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted corpus is visible in the card detail, with a list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `draftStep` completes. The card transitions to `DRAFTING` then `DRAFT_READY`. The draft appears: the full body text, a fidelity score (0–100 indicating how closely it matches the voice owner's style markers), and a list of style tokens the agent detected and preserved.
6. If the guardrail passes, the status is `DRAFT_READY`. If the guardrail rejected one or more iterations, the card shows a `guardrailRejections` count so the user knows how many retries were needed.
7. The user can submit another brief; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DraftEndpoint` | `HttpEndpoint` | `/api/drafts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `DraftEntity`, `DraftView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DraftEntity` | `EventSourcedEntity` | Per-draft lifecycle: submitted → sanitized → drafting → draft-ready → failed. Source of truth. | `DraftEndpoint`, `CorpusSanitizer`, `DraftWorkflow` | `DraftView` |
| `CorpusSanitizer` | `Consumer` | Subscribes to `BriefSubmitted` events; redacts PII from writing-sample attachments; calls `DraftEntity.attachSanitizedCorpus`. | `DraftEntity` events | `DraftEntity` |
| `DraftWorkflow` | `Workflow` | One workflow per draft. Steps: `awaitSanitizedStep` → `draftStep` → error. | started by `CorpusSanitizer` once sanitized event lands | `GhostwriterAgent`, `DraftEntity` |
| `GhostwriterAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the writing brief as task instructions and the sanitized corpus samples as task attachments; returns `DraftResult`. | invoked by `DraftWorkflow` | returns draft |
| `DraftOutputGuardrail` | supporting class | Validates the agent's candidate draft: no verbatim reproduction, no leaked redaction markers, required fields present, fidelity score in range. Registered on `GhostwriterAgent` as an `after-llm-response` hook. | `GhostwriterAgent` response | pass-through or rejection |
| `DraftView` | `View` | Read model: one row per draft for the UI. | `DraftEntity` events | `DraftEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record VoiceSample(
    String sampleId,
    String title,
    String rawBody,
    String format   // blog-post | memo | release-note | social
) {}

record WritingBrief(
    String draftId,
    String voiceOwnerId,
    String topic,
    String format,      // blog-post | memo | release-note | social
    int targetWordCount,
    List<VoiceSample> samples,
    String requestedBy,
    Instant submittedAt
) {}

record SanitizedCorpus(
    List<SanitizedSample> samples,
    List<String> piiCategoriesFound
) {}

record SanitizedSample(
    String sampleId,
    String redactedBody
) {}

record StyleMarker(
    String token,           // e.g. "Oxford comma", "em-dash preference", "first-person plural"
    String evidence         // short quote from source corpus
) {}

record DraftResult(
    String draftBody,
    int fidelityScore,          // 0..100
    List<StyleMarker> markers,
    int guardrailRejections,    // how many iterations were rejected before a clean draft landed
    Instant generatedAt
) {}

record Draft(
    String draftId,
    Optional<WritingBrief> brief,
    Optional<SanitizedCorpus> sanitizedCorpus,
    Optional<DraftResult> result,
    DraftStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DraftStatus {
    SUBMITTED, SANITIZED, DRAFTING, DRAFT_READY, FAILED
}
```

Events on `DraftEntity`: `BriefSubmitted`, `CorpusSanitized`, `DraftingStarted`, `DraftReady`, `DraftFailed`.

Every nullable lifecycle field on the `Draft` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/drafts` — body `{ voiceOwnerId, topic, format, targetWordCount, samples: [VoiceSample], requestedBy }` → `{ draftId }`.
- `GET /api/drafts` — list all drafts, newest-first.
- `GET /api/drafts/{id}` — one draft.
- `GET /api/drafts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Ghostwriter</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted drafts (status pill + format chip + age) and a right pane with the selected draft's detail — writing brief summary, sanitized corpus preview, draft body, fidelity score bar, style markers list, and guardrail-rejections count.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `CorpusSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from every writing sample before any LLM call. Records which categories were found.
- **G1 — after-llm-response guardrail**: runs on every turn of `GhostwriterAgent`. Asserts the candidate draft contains no verbatim runs ≥ 40 characters taken from the source corpus (voice-impersonation check), that no `[REDACTED-*]` tokens from the sanitizer appear in the output, that `fidelityScore` is in 0–100, and that `draftBody` is non-empty. On failure, returns a structured rejection to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `GhostwriterAgent` → `prompts/ghostwriter.md`. The single decision-making LLM. System prompt instructs it to read the attached writing samples, identify style markers, and draft new content for the given brief that reproduces the voice owner's patterns without copying their exact phrasing.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a blog-post brief for a seeded voice owner; within 30 s a draft appears with a fidelity score ≥ 60, at least two style markers, and a guardrail-rejections count of 0.
2. **J2** — The agent's first response on a brief contains a verbatim run ≥ 40 chars copied from the source corpus (mock LLM path) — the `after-llm-response` guardrail rejects it; the second iteration produces a clean draft; the UI shows `guardrailRejections: 1`.
3. **J3** — A writing sample containing `sarah.chen@corp.example` and `EIN 94-XXXXXXX` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-EIN]`; the entity's `brief.samples[].rawBody` retains the raw text for audit.
4. **J4** — If the agent exhausts its iteration budget without producing a passing draft, the entity transitions to `FAILED` and the UI shows the final guardrail rejection reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ghostwriter demonstrating the single-agent × content-editorial cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-content-editorial-voice-ghostwriter. Java package io.akka.samples.ghostwriter.
Akka 3.6.0. HTTP port 9714.

Components to wire (exactly):

- 1 AutonomousAgent GhostwriterAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/ghostwriter.md>) and
  .capability(TaskAcceptance.of(GENERATE_DRAFT).maxIterationsPerTask(3)). The task receives
  the writing brief as its instruction text and each sanitized writing sample as a separate
  task ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes)
  is the canonical call). Output: DraftResult{draftBody: String, fidelityScore: int (0..100),
  markers: List<StyleMarker>, guardrailRejections: int, generatedAt: Instant}. The agent is
  configured with an after-llm-response guardrail (see G1 in eval-matrix.yaml) registered via
  the agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  the response within its 3-iteration budget.

- 1 Workflow DraftWorkflow per draftId with two steps plus error:
  * awaitSanitizedStep — polls DraftEntity.getDraft every 1s; on draft.sanitizedCorpus()
    .isPresent() advances to draftStep. WorkflowSettings.stepTimeout 15s.
  * draftStep — emits DraftingStarted, then calls componentClient.forAutonomousAgent(
    GhostwriterAgent.class, "ghostwriter-" + draftId).runSingleTask(
      TaskDef.instructions(formatBrief(draft.brief))
        .attachment("sample-1.txt", sample1Bytes)
        .attachment("sample-2.txt", sample2Bytes)
        ...  // one attachment per sanitized sample
    ) — returns a taskId, then forTask(taskId).result(GENERATE_DRAFT) to fetch the result.
    On success calls DraftEntity.recordDraft(result). WorkflowSettings.stepTimeout 90s
    with defaultStepRecovery maxRetries(2).failoverTo(DraftWorkflow::error).
  * error step transitions the entity to FAILED with a reason string.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity DraftEntity (one per draftId). State Draft{draftId: String,
  brief: Optional<WritingBrief>, sanitizedCorpus: Optional<SanitizedCorpus>,
  result: Optional<DraftResult>, status: DraftStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. DraftStatus enum: SUBMITTED, SANITIZED, DRAFTING,
  DRAFT_READY, FAILED. Events: BriefSubmitted{brief}, CorpusSanitized{sanitizedCorpus},
  DraftingStarted{}, DraftReady{result}, DraftFailed{reason}. Commands: submit,
  attachSanitizedCorpus, markDrafting, recordDraft, fail, getDraft. emptyState() returns
  Draft.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer CorpusSanitizer subscribed to DraftEntity events; on BriefSubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, EIN/SSN-like, payment-card-like,
  postal addresses, person-name heuristic, account-id-like tokens) over each sample's rawBody,
  computes the list of categories found, builds SanitizedCorpus, then calls
  DraftEntity.attachSanitizedCorpus(sanitizedCorpus). After that lands, starts a
  DraftWorkflow with id = "draft-" + draftId.

- 1 View DraftView with row type DraftRow (mirrors Draft minus brief.samples[].rawBody — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes DraftEntity events. ONE query getAllDrafts: SELECT * AS drafts FROM draft_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * DraftEndpoint at /api with POST /drafts (body
    {voiceOwnerId, topic, format, targetWordCount,
    samples: [{sampleId, title, rawBody, format}], requestedBy};
    mints draftId; calls DraftEntity.submit; returns {draftId}), GET /drafts
    (list from getAllDrafts, sorted newest-first), GET /drafts/{id} (one row), GET
    /drafts/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- DraftTasks.java declaring one Task<R> constant: GENERATE_DRAFT = Task.name("Generate draft")
  .description("Read the attached writing samples and produce a DraftResult matching the voice
  owner's style for the given brief").resultConformsTo(DraftResult.class). DO NOT skip this —
  the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records VoiceSample, WritingBrief, SanitizedCorpus, SanitizedSample, StyleMarker,
  DraftResult, Draft, DraftStatus.

- DraftOutputGuardrail.java implementing the after-llm-response hook. Reads the candidate
  DraftResult from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9714 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The GhostwriterAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/voice-samples.jsonl with 3 seeded voice-owner corpora:
  "alex-chen" (product manager blog-post style, 3 samples ~400 words each),
  "morgan-riley" (engineering changelog memo style, 3 samples ~250 words each),
  "jordan-park" (social media / thought-leadership style, 3 samples ~120 words each).
  Each corpus contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/sample-events/seed-briefs.jsonl with 3 paired brief examples:
  a product-launch blog post for alex-chen, an internal release-note memo for morgan-riley,
  and a conference-recap social post for jordan-park.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = generative-assist
  (the agent's output requires human edit before publication), oversight.human_in_loop = true,
  failure.failure_modes including "verbatim-reproduction", "voice-mismatch",
  "pii-leakage-via-llm", "low-fidelity-draft"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/ghostwriter.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Ghostwriter", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of draft cards; right = selected-draft detail with writing brief summary,
  sanitized corpus preview, draft body, fidelity score bar, style markers list, and
  guardrail-rejections count).
  Browser title exactly: <title>Akka Sample: Ghostwriter</title>. No subtitle on the
  Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(draftId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-draft.json — 6 DraftResult entries covering all three voice owners.
      Each entry has a non-empty draftBody (~200 words), a fidelityScore between 55 and 95,
      a markers list with 2–4 StyleMarker entries, and guardrailRejections: 0. Plus 2
      deliberately FAILING entries (one with a verbatim run ≥ 40 chars copied from the
      seeded corpus; one with a [REDACTED-EMAIL] token in the draftBody) — the guardrail
      blocks both, exercising the retry path. The mock should select a failing entry on
      the FIRST iteration of every 3rd draft (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(draftId) helper makes per-draft selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. GhostwriterAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DraftTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (draftStep
  90s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on the Draft row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: DraftTasks.java with GENERATE_DRAFT = Task.name(...).description(...)
  .resultConformsTo(DraftResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9714 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (GhostwriterAgent).
  DraftOutputGuardrail is a validation hook, not an agent.
- The writing samples are passed as Task ATTACHMENTS, never inlined into the agent's
  instructions. Verify the generated draftStep uses TaskDef.attachment(...) per sample
  and not string interpolation into the instruction text.
- The after-llm-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external post-processing step. Lesson 1's AutonomousAgent contract
  is the authoritative reference for how the hook is registered.
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
