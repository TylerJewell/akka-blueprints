# SPEC — morning-email-debrief

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Morning Email Debrief.
**One-line pitch:** A scheduled worker reads the morning mailbox, redacts PII, summarises each email with AI, and assembles a structured debrief digest ready to read.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`EmailSummarizerAgent` and `DebriefAssemblerAgent`). Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw email event and any LLM call — so the model never sees personal identifiers from the mailbox.
- An **eval-periodic** sampler running every 60 minutes scores completed debriefs for coverage (did all important emails make it into the digest?) and tone (is the language appropriate for an executive audience?).

The result is a repeatable, auditable morning-briefing pipeline where AI summarisation is governed end-to-end: no raw PII reaches the model, and output quality is continuously measured.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the current debrief run: a list of emails received, their sanitized subjects, per-email summaries, and (once assembled) the full digest.
2. `MailboxPoller` (TimedAction) ticks every 60 s in dev mode and inserts simulated emails into `EmailQueue`. (A `RequestSimulator` style — drips canned email lines.)
3. For each new email: `EmailPiiSanitizer` (Consumer) redacts the payload, then `EmailSummarizerAgent` produces a `DebriefEntry`.
4. Once a batch is complete (all emails in the run are summarised), `DebriefWorkflow` calls `DebriefAssemblerAgent` to produce the `MorningDebrief` digest. The debrief transitions to READY.
5. The digest is displayed in the UI — subject lines, per-email priority chips, and the assembled narrative.
6. `EvalRunner` (TimedAction) ticks every 60 minutes, picks READY debriefs without an `evalScore`, calls `DebriefEvalAgent`, and writes a score back via a `DebriefEvalScored` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MailboxPoller` | `TimedAction` | Drips simulated emails into `EmailQueue` every 60 s. | scheduler | `EmailQueue` |
| `EmailQueue` | `EventSourcedEntity` | Append-only log of `EmailReceived` events per run. | `MailboxPoller`, `DebriefEndpoint` | `EmailPiiSanitizer` |
| `EmailPiiSanitizer` | `Consumer` | Reads `EmailReceived` events, applies regex+heuristic redaction, emits `EmailSanitized` via `EmailEntity`. | `EmailQueue` events | `EmailEntity` |
| `EmailSummarizerAgent` | `Agent` (typed, NOT autonomous) | Produces a `DebriefEntry` from a sanitized email payload. | invoked by `DebriefWorkflow` | returns `DebriefEntry` |
| `DebriefAssemblerAgent` | `AutonomousAgent` | Assembles all per-email `DebriefEntry` values into a `MorningDebrief` digest. | invoked by `DebriefWorkflow` | returns `MorningDebrief` |
| `DebriefWorkflow` | `Workflow` | Per-run orchestration: wait for all emails sanitized → summarize each → assemble → finalize. | `EmailPiiSanitizer` (one workflow per run id) | `EmailEntity`, `DebriefEntity` |
| `EmailEntity` | `EventSourcedEntity` | Lifecycle per email: received → sanitized → summarised. | `DebriefWorkflow` | `EmailView` |
| `DebriefEntity` | `EventSourcedEntity` | Lifecycle per debrief run: created → assembling → ready. | `DebriefWorkflow` | `DebriefView` |
| `EmailView` | `View` | Read-model row per email for the UI. | `EmailEntity` events | `DebriefEndpoint` |
| `DebriefView` | `View` | Read-model row per debrief run. | `DebriefEntity` events | `DebriefEndpoint` |
| `EvalRunner` | `TimedAction` | Every 60 min, samples READY debriefs without scores; calls eval agent; writes `DebriefEvalScored`. | scheduler | `DebriefEntity` |
| `DebriefEndpoint` | `HttpEndpoint` | `/api/debrief/*` and `/api/email/*` — list, get, SSE. | — | `DebriefView`, `EmailView`, `DebriefEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record ReceivedEmail(String emailId, String runId, String from, String subject, String body, Instant receivedAt) {}

record SanitizedEmailPayload(String redactedSubject, String redactedBody, List<String> piiCategoriesFound) {}

record DebriefEntry(String emailId, String oneLinerSummary, Priority priority, String category) {}
enum Priority { HIGH, MEDIUM, LOW }

record MorningDebrief(String runId, String narrativeSummary, List<DebriefEntry> entries, int totalEmailCount, Instant assembledAt) {}

record EvalResult(Integer score, String rationale) {}

record EmailRecord(
    String emailId,
    String runId,
    ReceivedEmail incoming,
    Optional<SanitizedEmailPayload> sanitized,
    Optional<DebriefEntry> entry,
    EmailStatus status,
    Instant createdAt
) {}

enum EmailStatus { RECEIVED, SANITIZED, SUMMARISED }

record DebriefRun(
    String runId,
    int totalEmails,
    int summarisedCount,
    Optional<MorningDebrief> debrief,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    DebriefStatus status,
    Instant createdAt,
    Optional<Instant> readyAt
) {}

enum DebriefStatus { CREATED, ASSEMBLING, READY, FAILED }
```

Events on `EmailEntity`: `EmailReceived`, `EmailSanitized`, `EmailSummarised`.

Events on `DebriefEntity`: `DebriefCreated`, `DebriefAssembling`, `DebriefReady`, `DebriefFailed`, `DebriefEvalScored`.

Events on `EmailQueue`: `EmailReceived` (raw audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/debrief` — list all debrief runs. Optional `?status=…`.
- `GET /api/debrief/{runId}` — one debrief run.
- `GET /api/debrief/{runId}/emails` — all emails for a run.
- `GET /api/email/{emailId}` — one email record.
- `GET /api/debrief/sse` — Server-Sent Events for debrief and email changes.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Morning Email Debrief</title>`.

App UI tab is the most distinctive: it shows the **current debrief run** with two panels — a left email list with priority chips and status badges, and a right digest panel showing the assembled narrative plus per-email entries.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `EmailPiiSanitizer` Consumer): redacts email addresses, phone numbers, names, account IDs, and other personal identifiers from the email body and subject before any LLM sees them. Records the categories found for audit.
- **E1 — eval-periodic** (every 60 minutes): scores completed debriefs on coverage (were high-priority emails represented?) and tone (is the digest language suitable for the intended audience?).

## 9. Agent prompts

- `EmailSummarizerAgent` → `prompts/email-summarizer.md`. Typed summarizer. Returns exactly one `DebriefEntry` per email.
- `DebriefAssemblerAgent` → `prompts/debrief-assembler.md`. Assembles all entries into one `MorningDebrief` digest.
- `DebriefEvalAgent` (used by `EvalRunner`) → `prompts/debrief-eval.md`. Scores a completed debrief on a 1–5 rubric.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Poller drips a batch of emails; all appear in the UI within 60 s; each passes sanitize → summarize → debrief assembly → READY.
2. **J2** — Completed debrief contains no raw PII in the assembled narrative (audit check via log inspection).
3. **J3** — EvalRunner scores a READY debrief within 60 minutes; the score and rationale appear in the UI.
4. **J4** — SSE stream delivers live status updates as each email progresses through the pipeline.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named morning-email-debrief demonstrating the continuous-monitor × ops-automation
cell. Runs out of the box (in-memory mailbox source; no real email integration). Maven group
io.akka.samples, artifact continuous-monitor-ops-automation-morning-email-debrief. Java package
io.akka.samples.workflowmorningemaildebrief. HTTP port 9855.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) EmailSummarizerAgent — summarizer. System prompt loaded from
  prompts/email-summarizer.md. Input: SanitizedEmailPayload{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: DebriefEntry{emailId: String, oneLinerSummary:
  String, priority: Priority (enum HIGH/MEDIUM/LOW), category: String}. Defaults to priority HIGH
  under uncertainty. Category is a short freeform label (e.g. "billing", "legal", "ops").

- 1 AutonomousAgent DebriefAssemblerAgent — definition() with capability(TaskAcceptance.of(ASSEMBLE)
  .maxIterationsPerTask(3)). System prompt from prompts/debrief-assembler.md. Input: List<DebriefEntry>
  + runId: String + totalEmailCount: int. Output: MorningDebrief{runId, narrativeSummary, entries:
  List<DebriefEntry>, totalEmailCount, assembledAt}. The agent does NOT send any external call.

- 1 AutonomousAgent DebriefEvalAgent — definition() with capability(TaskAcceptance.of(EVAL_DEBRIEF)
  .maxIterationsPerTask(2)). System prompt from prompts/debrief-eval.md. Input: MorningDebrief.
  Output: EvalResult{score: Integer 1–5, rationale: String}.

- 1 Workflow DebriefWorkflow per run with steps: awaitAllSanitizedStep -> summarizeAllStep ->
  assembleStep -> finalizeStep. awaitAllSanitizedStep polls EmailView every 10 s until all emails
  for the run have status SUMMARISED or the run's totalEmails count is reached. summarizeAllStep
  iterates the sanitized emails, calling EmailSummarizerAgent for each with stepTimeout 15s.
  assembleStep calls DebriefAssemblerAgent with stepTimeout 60s. finalizeStep emits DebriefReady
  or DebriefFailed. WorkflowSettings.builder().stepTimeout applies per step as above.

- 3 EventSourcedEntities:
  * EmailQueue — append-only audit log of inbound emails per run. Command receive(ReceivedEmail)
    emits EmailReceived{incoming}.
  * EmailEntity (one per emailId) — per-email lifecycle. State EmailRecord{emailId, runId,
    incoming: ReceivedEmail{emailId, runId, from, subject, body, receivedAt},
    Optional<SanitizedEmailPayload> sanitized, Optional<DebriefEntry> entry,
    EmailStatus status, Instant createdAt}. EmailStatus enum: RECEIVED, SANITIZED, SUMMARISED.
    Events: EmailReceived, EmailSanitized, EmailSummarised.
    Commands: registerIncoming, attachSanitized, attachEntry, getEmail. emptyState() returns
    EmailRecord.initial("", "") without commandContext() reference.
  * DebriefEntity (one per runId) — per-run lifecycle. State DebriefRun{runId, totalEmails,
    summarisedCount, Optional<MorningDebrief> debrief, Optional<Integer> evalScore,
    Optional<String> evalRationale, DebriefStatus status, Instant createdAt,
    Optional<Instant> readyAt}. DebriefStatus enum: CREATED, ASSEMBLING, READY, FAILED.
    Events: DebriefCreated, DebriefAssembling, DebriefReady, DebriefFailed, DebriefEvalScored.
    Commands: create, markAssembling, markReady, markFailed, recordEval, getDebrief.
    emptyState() returns DebriefRun.initial("") without commandContext() reference.

- 1 Consumer EmailPiiSanitizer subscribed to EmailQueue events; for each EmailReceived, applies a
  regex+heuristic redaction pipeline (email addresses, phone numbers, names, account-like tokens,
  national ID patterns) to subject + body, builds SanitizedEmailPayload with piiCategoriesFound,
  and calls EmailEntity.registerIncoming followed by attachSanitized. Then increments the
  DebriefEntity summarisedCount (or creates it via DebriefEntity.create if first email for the run).
  Starts or joins a DebriefWorkflow with runId as the workflow id.

- 2 Views:
  * EmailView with row type EmailRow (mirrors EmailRecord minus raw incoming body). Table updater
    consumes EmailEntity events. ONE query getAllEmailsForRun(runId) SELECT * FROM email_view
    WHERE run_id = :runId. Optional<T> for every nullable field on the row record.
  * DebriefView with row type DebriefRow (mirrors DebriefRun minus the full MorningDebrief body,
    keeping a brief summary for list display). Table updater consumes DebriefEntity events.
    ONE query getAllDebriefs() SELECT * FROM debrief_view.

- 2 TimedActions:
  * MailboxPoller — every 60s (configurable via MAILBOX_POLL_SECONDS env var), reads next batch
    from src/main/resources/sample-events/email-batches.jsonl and calls EmailQueue.receive for
    each email in the batch, all under the same generated runId.
  * EvalRunner — every 60 minutes (configurable via EVAL_RUNNER_SECONDS for testing), queries
    DebriefView.getAllDebriefs(), picks up to 3 READY debriefs without an evalScore (oldest-first),
    calls DebriefEvalAgent with the MorningDebrief, then calls DebriefEntity.recordEval per run.

- 2 HttpEndpoints:
  * DebriefEndpoint at /api with GET /debrief, GET /debrief/{runId},
    GET /debrief/{runId}/emails, GET /email/{emailId}, GET /debrief/sse, and
    GET /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DebriefTasks.java declaring two Task<R> constants: ASSEMBLE (MorningDebrief),
  EVAL_DEBRIEF (EvalResult).
- Domain records ReceivedEmail, SanitizedEmailPayload, DebriefEntry, MorningDebrief, EvalResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9855 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/email-batches.jsonl with 3 canned batch lines, each
  batch containing 3–5 emails covering HIGH/MEDIUM/LOW priority situations (legal notice,
  daily report digest, newsletter, ops alert, routine update).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, E1 eval-periodic
  performance-monitor. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = read-only-summary,
  oversight.human_in_loop = false, failure.failure_modes including "pii-leakage-via-llm" and
  "missing-high-priority-email-from-digest"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/email-summarizer.md, prompts/debrief-assembler.md, prompts/debrief-eval.md loaded as
  agent system prompts.
- README.md at the project root: title "Akka Sample: Morning Email Debrief", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no npm).
  Five tabs matching the formal exemplar with the App UI tab using a two-panel layout
  (left = email list with priority chips and status badges; right = assembled debrief digest
  with per-entry cards and eval score chip when available). Browser title exactly:
  <title>Akka Sample: Morning Email Debrief</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    email-summarizer.json — 8–12 DebriefEntry entries spanning HIGH
      (legal notice, ops alert, executive request), MEDIUM (project update,
      customer feedback), and LOW (newsletter, automated digest, FYI thread).
      Each entry has a one-liner summary and a category label.
    debrief-assembler.json — 2–3 MorningDebrief entries with narrativeSummary
      of 3–5 sentences summarising the day's top themes. Entries list mirrors
      the input list. No invented names, no echoed [REDACTED] tokens.
    debrief-eval.json — 6–8 EvalResult entries with score 1–5 and one-sentence
      rationales matching the rubric (coverage / tone / completeness).
- A MockModelProvider.seedFor(runId) helper makes per-run selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- EmailPiiSanitizer runs INSIDE a Consumer before any LLM call — not inside an Agent's
  prompt and not after the LLM has seen the raw email payload.
- The generated static-resources/index.html must include the mermaid CSS overrides AND
  theme variables from Lesson 24 (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state names render
  black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel attribute,
  NEVER by NodeList index (Lesson 26). No "hidden" zombie panels in the DOM — delete removed
  tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
