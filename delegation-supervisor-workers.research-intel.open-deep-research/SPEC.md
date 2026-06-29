# SPEC — open-deep-research

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Open Deep Research.
**One-line pitch:** Type a research question; a manager agent delegates web-browsing and document-reading to two retrieval workers running in parallel, then synthesises a cited answer from their outputs.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans retrieval work out to two AutonomousAgents in parallel, gathers their outputs, sanitizes PII, and asks a third AutonomousAgent to synthesise a cited answer. The blueprint also demonstrates a **before-tool-call guardrail** that enforces a URL allow-list on every browser tool invocation, a **PII sanitizer** that scrubs fetched content before synthesis, an **eval-event** control that samples the manager's synthesis decision for quality, and a **human-oversight** (deployer-runtime-monitoring) flag exposed via a monitoring endpoint.

## 3. User-facing flows

The user opens the App UI tab and submits a research question via the form.

1. The system creates a `ResearchRun` record in `QUEUED` and starts a `ResearchWorkflow`.
2. The manager decomposes the question into two parallel retrieval tasks: a URL list for `WebBrowsingAgent` and a document reference for `FileReaderAgent`.
3. Before each browser tool call, a `before-tool-call` guardrail checks the URL against the allow-list. Disallowed URLs are rejected immediately; the retrieval step records the block and continues with remaining URLs.
4. The workflow forks: both workers run concurrently. Each returns a typed payload.
5. A PII sanitizer pass strips any personal data from the retrieved content before it enters the synthesis prompt.
6. The manager merges the two payloads into a `SynthesisedAnswer { summary, citations, piiSanitized, synthesisedAt }`.
7. If either worker times out after 60 seconds, the workflow short-circuits: the manager synthesises from whichever side returned, and the run enters `DEGRADED`.
8. A completed `ANSWERED` run is available for eval sampling; the per-step quality score appears in the App UI alongside the answer.

A `QuestionSimulator` (TimedAction) drips a sample question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchManager` | `AutonomousAgent` | Decomposes the question into a retrieval plan; synthesises the merged content into a cited answer. | `ResearchWorkflow` | returns typed result to workflow |
| `WebBrowsingAgent` | `AutonomousAgent` | Fetches text from allowed URLs; pre-call guardrail enforces the URL allow-list. | `ResearchWorkflow` | — |
| `FileReaderAgent` | `AutonomousAgent` | Reads and parses structured documents; returns extracted sections. | `ResearchWorkflow` | — |
| `ResearchWorkflow` | `Workflow` | Coordinates the parallel retrieval fan-out, PII sanitization, synthesis, and monitoring flag. | `ResearchEndpoint`, `QuestionConsumer` | `ResearchRunEntity` |
| `ResearchRunEntity` | `EventSourcedEntity` | Holds the run's lifecycle (queued → in-progress → answered / degraded / blocked). | `ResearchWorkflow` | `ResearchView` |
| `QuestionQueue` | `EventSourcedEntity` | Logs each submitted question for replay/audit. | `ResearchEndpoint`, `QuestionSimulator` | `QuestionConsumer` |
| `ResearchView` | `View` | List-of-runs read model. | `ResearchRunEntity` events | `ResearchEndpoint` |
| `QuestionConsumer` | `Consumer` | Listens to `QuestionQueue` events and starts a workflow per submission. | `QuestionQueue` events | `ResearchWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample question every 60 s. | scheduler | `QuestionQueue` |
| `EvalSampler` | `TimedAction` | Samples one answered run every 5 minutes for quality scoring; emits a `RunEvalScored` event. | scheduler | `ResearchRunEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/research/*` — submit, get, list, SSE, monitoring. | — | `ResearchView`, `QuestionQueue`, `ResearchRunEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QuestionRequest(String question, String requestedBy) {}

record RetrievalPlan(List<String> urlsToFetch, String documentReference) {}

record WebPage(String url, String title, String extractedText, Instant fetchedAt) {}
record WebBundle(List<WebPage> pages, boolean piiSanitized, Instant gatheredAt) {}

record DocumentSection(String heading, String content) {}
record DocumentBundle(String documentRef, List<DocumentSection> sections,
                      boolean piiSanitized, Instant readAt) {}

record Citation(String text, String sourceUrl, String sourceDocument) {}
record SynthesisedAnswer(String summary, List<Citation> citations,
                         boolean piiSanitized, Instant synthesisedAt) {}

record ResearchRun(
    String runId,
    String question,
    RunStatus status,
    Optional<WebBundle> webContent,
    Optional<DocumentBundle> docContent,
    Optional<SynthesisedAnswer> answer,
    Optional<String> blockedUrl,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    boolean oversightFlagged,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus { QUEUED, IN_PROGRESS, ANSWERED, DEGRADED, BLOCKED }
```

### Events (on `ResearchRunEntity`)

`RunCreated`, `WebContentAttached`, `DocContentAttached`, `RunAnswered`, `RunDegraded`, `RunBlocked`, `RunEvalScored`, `OversightFlagged`.

### Events (on `QuestionQueue`)

`QuestionSubmitted { runId, question, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/research` — body `{ question }` → `{ runId }`. Starts a workflow.
- `GET /api/research` — list all runs. Optional `?status=QUEUED|IN_PROGRESS|ANSWERED|DEGRADED|BLOCKED`.
- `GET /api/research/{id}` — one run.
- `GET /api/research/sse` — server-sent events stream of every run change.
- `GET /api/research/monitoring` — returns JSON oversight summary (flagged run count, last flag time).
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Open Deep Research"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a question, live list of runs with status pills, expand-row to see web pages, document sections, synthesised answer, citations, and eval score.

Browser title: `<title>Akka Sample: Open Deep Research</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `WebBrowsingAgent`): checks each URL against the allow-list before the browser tool executes. Blocking per URL; the step continues with remaining allowed URLs. Disallowed URL recorded as `blockedUrl` on the run.
- **S1 — PII sanitizer** (`pii`): runs over both `WebBundle.extractedText` and `DocumentBundle.sections` before content enters the synthesis prompt in `ResearchWorkflow.sanitizeStep`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one answered run every 5 minutes and emits a `RunEvalScored` event with a 1–5 score and rationale.
- **H1 — human-oversight flag** (`deployer-runtime-monitoring`): `ResearchWorkflow` emits an `OversightFlagged` event on any run that exceeds the configured step-retry budget or returns a degraded status; `GET /api/research/monitoring` exposes the count.

## 9. Agent prompts

- `ResearchManager` → `prompts/manager.md`. Decomposes the question into a retrieval plan; later synthesises retrieved content into a cited answer.
- `WebBrowsingAgent` → `prompts/web-browsing-agent.md`. Fetches URLs and returns extracted text.
- `FileReaderAgent` → `prompts/file-reader-agent.md`. Reads a document reference and returns structured sections.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a question; run progresses QUEUED → IN_PROGRESS → ANSWERED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `WebBrowsingAgent` timeout to 1 s); run enters DEGRADED; manager synthesises from document content alone.
3. **J3** — Submit a question whose retrieval plan includes a disallowed URL; guardrail blocks the URL; run records `blockedUrl`; remaining URLs are still fetched.
4. **J4** — Wait for `EvalSampler`; the answered run gains a score visible in the App UI.
5. **J5** — Inject a retry-budget breach; `OversightFlagged` event emitted; monitoring endpoint reports the flag.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named open-deep-research demonstrating the
delegation-supervisor-workers × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-research-intel-open-deep-research.
Java package io.akka.samples.opendeepresearch. Akka 3.6.0. HTTP port 9206.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ResearchManager — definition() with capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/manager.md. Returns RetrievalPlan{urlsToFetch, documentReference} for PLAN
    and SynthesisedAnswer{summary, citations, piiSanitized, synthesisedAt} for SYNTHESISE.
  * WebBrowsingAgent — capability(TaskAcceptance.of(FETCH_WEB).maxIterationsPerTask(4)). System
    prompt from prompts/web-browsing-agent.md. Returns WebBundle{pages: List<WebPage{url, title,
    extractedText, fetchedAt}>, piiSanitized, gatheredAt}. Has a before-tool-call guardrail that
    checks each URL against the configured allow-list before the fetch tool executes; blocked URLs
    are skipped and recorded.
  * FileReaderAgent — capability(TaskAcceptance.of(READ_DOC).maxIterationsPerTask(3)). System
    prompt from prompts/file-reader-agent.md. Returns DocumentBundle{documentRef,
    sections: List<DocumentSection{heading, content}>, piiSanitized, readAt}.

- 1 Workflow ResearchWorkflow with steps:
  planStep -> [parallel] fetchWebStep, readDocStep -> sanitizeStep -> synthesiseStep -> monitorStep -> emitStep.
  planStep calls forAutonomousAgent(ResearchManager.class, PLAN).
  fetchWebStep and readDocStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(ResearchWorkflow::fetchWebStep, ofSeconds(60)) and
  stepTimeout(ResearchWorkflow::readDocStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with RunDegraded.
  sanitizeStep strips PII from both bundles before passing to synthesis; implemented as a
  deterministic text processor, not an LLM call.
  synthesiseStep calls forAutonomousAgent(ResearchManager.class, SYNTHESISE) with the sanitized
  inputs; give synthesiseStep a 90s stepTimeout.
  monitorStep checks the retry budget counter; if exceeded, calls ResearchRunEntity.flagOversight
  emitting OversightFlagged. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity ResearchRunEntity holding state ResearchRun{runId, question,
  RunStatus, Optional<WebBundle> webContent, Optional<DocumentBundle> docContent,
  Optional<SynthesisedAnswer> answer, Optional<String> blockedUrl,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, boolean oversightFlagged, Instant createdAt,
  Optional<Instant> finishedAt}. RunStatus enum: QUEUED, IN_PROGRESS, ANSWERED,
  DEGRADED, BLOCKED. Events: RunCreated, WebContentAttached, DocContentAttached,
  RunAnswered, RunDegraded, RunBlocked, RunEvalScored, OversightFlagged.
  Commands: createRun, attachWebContent, attachDocContent, answerRun, degradeRun,
  blockRun, recordEval, flagOversight, getRun.
  emptyState() returns ResearchRun.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity QuestionQueue with command enqueueQuestion(question, requestedBy)
  emitting QuestionSubmitted{runId, question, requestedBy, submittedAt}.

- 1 View ResearchView with row type ResearchRunRow (mirrors ResearchRun minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  ResearchRunEntity events. ONE query getAllRuns SELECT * AS runs FROM run_view.
  No WHERE status filter — caller filters client-side.

- 1 Consumer QuestionConsumer subscribed to QuestionQueue events; on QuestionSubmitted
  starts a ResearchWorkflow with the runId as the workflow id.

- 2 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-questions.jsonl and calls
    QuestionQueue.enqueueQuestion.
  * EvalSampler — every 5 minutes, queries ResearchView.getAllRuns, picks the oldest
    ANSWERED run without an evalScore, runs a 1–5 rubric judge over the synthesised
    answer, then calls ResearchRunEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /research, GET /research, GET /research/{id},
    GET /research/sse, GET /research/monitoring, and the /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ResearchTasks.java declaring four Task<R> constants: PLAN (RetrievalPlan), FETCH_WEB
  (WebBundle), READ_DOC (DocumentBundle), SYNTHESISE (SynthesisedAnswer).
- Domain records RetrievalPlan, WebPage, WebBundle, DocumentSection, DocumentBundle,
  Citation, SynthesisedAnswer.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9206 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-questions.jsonl with 8 canned question lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls (G1 before-tool-call guardrail,
  S1 pii sanitizer, E1 eval-event on-decision-eval, H1 hotl deployer-runtime-monitoring)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  web-research-synthesis, decisions.authority_level = recommend-only,
  data.data_classes.pii = true (web content may contain PII),
  capabilities.content_generation = true; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/manager.md, prompts/web-browsing-agent.md, prompts/file-reader-agent.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Open Deep Research", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Open Deep Research</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (manager.json,
  web-browsing-agent.json, file-reader-agent.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    manager.json — list of either RetrievalPlan or SynthesisedAnswer objects.
      4–6 RetrievalPlan entries (urlsToFetch list of 2–4 URLs + documentReference) and
      4–6 SynthesisedAnswer entries (each with an 80–150 word summary, 3–6 citations,
      piiSanitized = true).
    web-browsing-agent.json — 4–6 WebBundle entries, each with 2–4 WebPage entries
      whose url values are from the allow-list, extractedText is 2–4 sentences,
      piiSanitized = false.
    file-reader-agent.json — 4–6 DocumentBundle entries, each with 3–5 sections
      whose heading strings are plausible document headings and content is 2–3 sentences,
      piiSanitized = false.
- A MockModelProvider.seedFor(runId) helper makes the selection deterministic per
  run id so the same run produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s retrieval
  workers, 90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion ResearchTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9206 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
