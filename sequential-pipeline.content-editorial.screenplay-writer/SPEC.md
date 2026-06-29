# SPEC — screenplay-writer

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Screenplay Writer.
**One-line pitch:** The user pastes a conversational thread (newsgroup posts or an email exchange); the system redacts personal data, extracts characters and setting, writes dialogue, and returns a formatted screenplay.

## 2. What this blueprint demonstrates

A sequential pipeline: one workflow runs four ordered stages — sanitize, analyze, write dialogue, format — each handing its typed output to the next. The governance pattern pairs a deterministic PII sanitizer at the front (so no agent ever sees raw names or addresses) with a before-agent-response guardrail at the back (so the finished screenplay cannot leak personal data carried over from the source).

## 3. User-facing flows

1. The user pastes a conversational thread into the App UI and submits. The system returns a `jobId` and the job appears in `RECEIVED` state.
2. The pipeline redacts personal data; the job moves to `SANITIZED` and shows a redaction count.
3. The analyzer extracts characters, setting, synopsis, and tone; the job moves to `ANALYZED`.
4. The dialogue writer drafts scene dialogue; the job moves to `DIALOGUE_WRITTEN`.
5. The formatter renders standard screenplay structure and runs the final PII check; the job moves to `COMPLETED` (or `BLOCKED` if leaked PII is found).
6. The UI streams each transition and renders the finished screenplay.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ThreadIntakeEndpoint | HttpEndpoint | Accept thread submissions; serve job reads and metadata | client / UI | ScreenplayWorkflow, ScreenplayView |
| AppEndpoint | HttpEndpoint | Serve the static UI | browser | static-resources |
| ScreenplayWorkflow | Workflow | Run sanitize → analyze → dialogue → format in order | ThreadIntakeConsumer, ThreadIntakeEndpoint | ThreadAnalyzerAgent, DialogueWriterAgent, ScreenplayFormatterAgent, ScreenplayEntity |
| ThreadAnalyzerAgent | Agent | Extract characters, setting, synopsis, tone | ScreenplayWorkflow | — |
| DialogueWriterAgent | Agent | Generate scene dialogue from the analysis | ScreenplayWorkflow | — |
| ScreenplayFormatterAgent | Agent | Format the screenplay; final PII guardrail fires here | ScreenplayWorkflow | — |
| ScreenplayEntity | EventSourcedEntity | Hold one job's lifecycle state and events | ScreenplayWorkflow | ScreenplayView |
| ScreenplayView | View | CQRS read model over ScreenplayEntity events | ScreenplayEntity | ThreadIntakeEndpoint |
| InboundThreadQueue | EventSourcedEntity | Queue of submitted/simulated threads | ThreadIntakeEndpoint, ThreadSimulator | ThreadIntakeConsumer |
| ThreadIntakeConsumer | Consumer | Start one workflow per queued thread | InboundThreadQueue | ScreenplayWorkflow |
| ThreadSimulator | TimedAction | Drip canned threads every 30s | sample-events file | InboundThreadQueue |

Names matter. `/akka:specify` uses them verbatim.

## 5. Data model

Authoritative record set — see `reference/data-model.md` for the full field table.

`ScreenplayJob` is both the entity state and the view row. Lifecycle fields are `Optional<T>` (Lesson 6): `sanitizedText`, `redactionCount`, `characters`, `setting`, `synopsis`, `tone`, `dialogue`, `screenplay`, `piiFindings`, `sanitizedAt`, `analyzedAt`, `dialogueAt`, `completedAt`, `failureReason`.

Status enum `ScreenplayStatus`: `RECEIVED`, `SANITIZED`, `ANALYZED`, `DIALOGUE_WRITTEN`, `COMPLETED`, `BLOCKED`, `FAILED`.

Events: `ThreadReceived`, `ThreadSanitized`, `ThreadAnalyzed`, `DialogueWritten`, `ScreenplayCompleted`, `ScreenplayBlocked`, `JobFailed`.

Agent result records: `ThreadAnalysis(List<String> characters, String setting, String synopsis, String tone)`, `DialogueDraft(String dialogue)`, `Screenplay(String formattedText, boolean containsPii, List<String> piiFindings)`.

## 6. API contract

Inline surface; payload schemas in `reference/api-contract.md`.

```
POST /api/threads                 -> { jobId }
GET  /api/jobs ?status=...        -> { jobs: [ScreenplayJob, ...] }
GET  /api/jobs/{jobId}            -> ScreenplayJob
GET  /api/jobs/sse               -> Server-Sent Events of ScreenplayJob
GET  /api/metadata/eval-matrix    -> text/yaml
GET  /api/metadata/risk-survey    -> text/yaml
GET  /api/metadata/readme         -> text/markdown
GET  /                            -> 302 /app/index.html
GET  /app/{*path}                 -> static-resources/{*path}
```

## 7. UI

Five tabs — Overview / Architecture / Risk Survey / Eval Matrix / App UI — described in `reference/ui-mockup.md`. Browser title: `<title>Akka Sample: Screenplay Writer</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26). Mermaid diagrams on the Architecture tab carry the state-label CSS overrides and theme variables from Lesson 24. The App UI tab provides a thread textarea, a submit button, a live SSE job list, and a panel rendering the finished screenplay with characters and redaction count.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. Two controls:

- **S1 — sanitizer · pii:** a deterministic redactor runs as the workflow's first step, replacing names and email/postal addresses in the source thread before any agent reads it; the redaction count is recorded on the entity.
- **G1 — guardrail · before-agent-response · pii:** a guardrail on `ScreenplayFormatterAgent` inspects the formatted screenplay for personal data carried over from the source; on a hit the workflow records `ScreenplayBlocked` and the job ends in `BLOCKED`.

## 9. Agent prompts

- `prompts/thread-analyzer.md` — extract characters, setting, synopsis, and tone from the sanitized thread.
- `prompts/dialogue-writer.md` — turn the analysis into natural scene dialogue.
- `prompts/screenplay-formatter.md` — render the dialogue in standard screenplay format and self-report any residual personal data.

## 10. Acceptance

Inline; full journeys in `reference/user-journeys.md`.

- **J1 — Thread to screenplay.** Submit a thread; within ~60 s the job reaches `COMPLETED` with non-empty `screenplay`, a character list, and a redaction count.
- **J2 — Redaction before agents.** A thread containing a real name and email shows `redactionCount >= 1` at the `SANITIZED` transition; the analyzer's input contains no raw address.
- **J3 — Leak blocked.** A screenplay that reproduces source PII drives the job to `BLOCKED` with `piiFindings` populated; no `COMPLETED` event fires.
- **J4 — Background simulation.** With no UI interaction, `ThreadSimulator` seeds threads and each becomes its own `COMPLETED` job.

---

## 11. Implementation directives

The whole SPEC.md (Sections 1–11) is the input to `/akka:specify @SPEC.md`. Section 11 carries the Akka-specific details Sections 1–10 do not repeat.

```
Create a sample named screenplay-writer demonstrating the sequential-pipeline ×
content-editorial cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact screenplay-writer. Java package
io.akka.samples.screenplaywriter. Akka 3.6.0. HTTP port 9412.

Components to wire (exactly):
- 3 request/response Agents (extends akka.javasdk.agent.Agent — NEVER downgrade or
  upgrade the primitive). Each exposes one method returning Effect<R> built with
  effects().systemMessage(<prompts/*.md>).userMessage(...).responseAs(R.class).thenReply():
  - ThreadAnalyzerAgent.analyze(String sanitizedThread) -> ThreadAnalysis.
  - DialogueWriterAgent.write(ThreadAnalysis analysis) -> DialogueDraft.
  - ScreenplayFormatterAgent.format(DialogueDraft draft) -> Screenplay. This agent
    registers a before-agent-response guardrail (G1) that scans the formatted text
    for residual source PII and sets Screenplay.containsPii / piiFindings.
- 1 Workflow ScreenplayWorkflow with steps sanitizeStep -> analyzeStep ->
  dialogueStep -> formatStep. sanitizeStep runs a deterministic PiiSanitizer (S1,
  regex over emails, postal addresses, and the names captured from the thread
  headers) on the raw thread, records redaction count, calls
  ScreenplayEntity.recordSanitized. analyzeStep / dialogueStep / formatStep call
  the agents via componentClient.forAgent().inSession(jobId).method(...).invoke(...)
  and write the matching entity command. On Screenplay.containsPii == true,
  formatStep calls ScreenplayEntity.recordBlocked and ends; otherwise
  recordCompleted. Override settings() with stepTimeout(60s) on analyzeStep,
  dialogueStep, formatStep and defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity ScreenplayEntity holding a ScreenplayJob record with id,
  sourceLabel, ScreenplayStatus enum, and Optional lifecycle fields (sanitizedText,
  redactionCount, characters, setting, synopsis, tone, dialogue, screenplay,
  piiFindings, sanitizedAt, analyzedAt, dialogueAt, completedAt, failureReason).
  Events: ThreadReceived, ThreadSanitized, ThreadAnalyzed, DialogueWritten,
  ScreenplayCompleted, ScreenplayBlocked, JobFailed. Commands: receive,
  recordSanitized, recordAnalysis, recordDialogue, recordCompleted, recordBlocked,
  fail, getJob. emptyState() returns ScreenplayJob.initial("", "") with no
  commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundThreadQueue with one command enqueueThread(sourceLabel,
  rawText) emitting ThreadQueued.
- 1 View ScreenplayView with row type ScreenplayJob, table updater consuming
  ScreenplayEntity events. ONE query getAllJobs: SELECT * AS jobs FROM
  screenplay_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 1 Consumer ThreadIntakeConsumer subscribed to InboundThreadQueue events; on each
  event starts a ScreenplayWorkflow with a fresh UUID jobId.
- 1 TimedAction ThreadSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/threads.jsonl and calls
  InboundThreadQueue.enqueueThread).
- 2 HttpEndpoints: ThreadIntakeEndpoint at /api with threads (POST), jobs list
  (filter client-side from getAllJobs), single job, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- Records ThreadAnalysis(List<String> characters, String setting, String synopsis,
  String tone), DialogueDraft(String dialogue), Screenplay(String formattedText,
  boolean containsPii, List<String> piiFindings).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9412
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/threads.jsonl with 8 canned conversational
  threads (newsgroup-style and email-style), each containing at least one name and
  one email or postal address so the sanitizer has work to do.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with controls S1 and G1 plus a matching
  simplified_view. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): pitch, component inventory,
  matrix cell, integration descriptor, how to run, the five tabs, ASCII
  architecture diagram, API contract, license. No governance-mechanisms section,
  no configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML are acceptable. Five tabs: Overview, Architecture (mermaid),
  Risk Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark / yellow accent / Instrument Sans / dot-grid).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ThreadAnalyzerAgent -> ThreadAnalysis, DialogueWriterAgent -> DialogueDraft,
  ScreenplayFormatterAgent -> Screenplay; write
  src/main/resources/mock-responses/{thread-analyzer,dialogue-writer,
  screenplay-formatter}.json with 4-6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in the Claude session; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the REFERENCE —
  env-var name, file path, or secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; it never echoes captured key material.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: Agent primitive matches the spec verbatim; never silently swap to
  AutonomousAgent.
- Lesson 4: explicit stepTimeout on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable lifecycle field on the view row record.
- Lesson 7: an AutonomousAgent (not used here) would require a Tasks.java companion;
  these are request/response Agents, so no Tasks.java.
- Lesson 8: verify model names are current before locking them.
- Lesson 9: the run command is "/akka:build" (Claude Code slash command), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode http-port = 9412 in application.conf.
- Lesson 11: never render source-platform metadata anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: static-resources/index.html includes the mermaid state-label CSS
  overrides and theme variables (edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  NodeList index; no display:none zombie panels — delete removed tabs.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the mock option), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
