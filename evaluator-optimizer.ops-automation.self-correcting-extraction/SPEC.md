# SPEC — self-correcting-extraction

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Correcting Extraction Agent with Memory.
**One-line pitch:** Submit a raw document; an extraction agent pulls structured fields; a scorer grades the extraction against memory-grounded ground truth; the two iterate until the extraction passes or the loop hits its budget cap.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`ExtractionAgent`) and a reviewer agent (`ScorerAgent`), feeding each correction note back into the next extraction attempt until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every cycle's scored verdict for downstream quality measurement, and a **halt** (budget-cap) that ends the loop gracefully at the configured attempt ceiling without leaving the job in a degenerate state. Memory is maintained at two levels: a window buffer (the last N attempts carried on the workflow state) and a database memory (verified field values persisted on `MemoryEntity` and queried by the Scorer at the start of each cycle).

## 3. User-facing flows

The user opens the App UI tab and submits a raw document (free text or a pre-loaded sample).

1. The system creates an `ExtractionJob` record in `EXTRACTING` and starts an `ExtractionWorkflow`.
2. The Extraction Agent reads the document and returns a `FieldMap` — a structured set of key-value pairs (e.g., invoice number, date, total amount, vendor name).
3. The Scorer retrieves any matching ground-truth entries from `MemoryEntity` and grades the `FieldMap`. It returns either `PASS` with a confidence score, or `CORRECT` with a typed `CorrectionNotes` payload (up to four bullets naming the field, the extracted value, and the expected form).
4. On `PASS`, the workflow transitions the job to `VERIFIED`, writes the accepted field values back to `MemoryEntity` for future runs, and emits a terminal `EvalRecorded` event.
5. On `CORRECT`, the workflow records the attempt, the scorer's verdict, and the correction notes on the entity, then calls the Extraction Agent again with the correction notes and the window-buffer context (the last three attempts). The agent produces attempt #2.
6. If the loop reaches `budgetCap` (default 4) without a `PASS`, the budget-cap halt activates: the workflow ends with `BUDGET_EXHAUSTED`, the highest-confidence extraction is preserved on the entity along with every attempt and every correction note for audit, and a terminal `EvalRecorded` event captures the outcome.

A `DocumentSimulator` (TimedAction) drips a canned document every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ExtractionAgent` | `AutonomousAgent` | Extracts structured fields from a raw document; accepts prior correction notes and window-buffer context on revision calls. | `ExtractionWorkflow` | returns `FieldMap` to workflow |
| `ScorerAgent` | `AutonomousAgent` | Grades a `FieldMap` against memory-grounded ground truth; returns `PASS` or `CORRECT` with notes. | `ExtractionWorkflow` | returns `ScorerVerdict` to workflow |
| `ExtractionWorkflow` | `Workflow` | Runs the extract → score → correct loop; halts at the budget ceiling. | `ExtractionEndpoint`, `DocumentConsumer` | `ExtractionJobEntity`, `MemoryEntity` |
| `ExtractionJobEntity` | `EventSourcedEntity` | Holds the job lifecycle, every attempt's field map, every scorer verdict, and the final outcome. | `ExtractionWorkflow` | `JobsView` |
| `MemoryEntity` | `EventSourcedEntity` | Persists verified field-value pairs keyed by document type; queried by the Scorer at each cycle. | `ExtractionWorkflow` (write on PASS), `ScorerAgent` (read via endpoint) | `ExtractionEndpoint` |
| `DocumentQueue` | `EventSourcedEntity` | Logs each submitted document for replay and audit. | `ExtractionEndpoint`, `DocumentSimulator` | `DocumentConsumer` |
| `JobsView` | `View` | List-of-jobs read model. | `ExtractionJobEntity` events | `ExtractionEndpoint` |
| `DocumentConsumer` | `Consumer` | Subscribes to `DocumentQueue` events; starts a workflow per submission. | `DocumentQueue` events | `ExtractionWorkflow` |
| `DocumentSimulator` | `TimedAction` | Drips a sample document every 60 s from `sample-events/sample-documents.jsonl`. | scheduler | `DocumentQueue` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `JobsView`, records an `EvalRecorded` event for any cycle that completed since the last tick. | scheduler | `ExtractionJobEntity` |
| `ExtractionEndpoint` | `HttpEndpoint` | `/api/jobs/*` — submit, get, list, SSE; `/api/memory/*`; plus `/api/metadata/*`. | — | `JobsView`, `DocumentQueue`, `ExtractionJobEntity`, `MemoryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record DocumentSubmission(String documentType, String rawText, String submittedBy) {}

record FieldMap(Map<String, String> fields, double confidence, Instant extractedAt) {}

record CorrectionNotes(List<String> bullets, String overallRationale) {}

record ScorerVerdict(
    ScorerDecision decision,
    CorrectionNotes notes,
    double confidence,
    Instant scoredAt
) {}

record ExtractionAttempt(
    int attemptNumber,
    FieldMap fieldMap,
    Optional<ScorerVerdict> verdict
) {}

record ExtractionJob(
    String jobId,
    String documentType,
    String rawText,
    int budgetCap,
    JobStatus status,
    List<ExtractionAttempt> attempts,
    Optional<Integer> verifiedAttemptNumber,
    Optional<FieldMap> verifiedFieldMap,
    Optional<String> exhaustionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum JobStatus { EXTRACTING, SCORING, VERIFIED, BUDGET_EXHAUSTED }

enum ScorerDecision { PASS, CORRECT }

record MemoryRecord(String documentType, String fieldName, String confirmedValue, Instant confirmedAt) {}
```

### Events (on `ExtractionJobEntity`)

`JobCreated`, `AttemptExtracted`, `AttemptScored`, `JobVerified`, `JobBudgetExhausted`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

### Events (on `MemoryEntity`)

`MemoryRecordWritten`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/jobs` — body `{ documentType, rawText, submittedBy? }` → `{ jobId }`. Starts a workflow.
- `GET /api/jobs` — list all jobs. Optional `?status=EXTRACTING|SCORING|VERIFIED|BUDGET_EXHAUSTED`.
- `GET /api/jobs/{id}` — one job (including every attempt and every verdict).
- `GET /api/jobs/sse` — server-sent events stream of every job change.
- `GET /api/memory/{documentType}` — returns the confirmed field-value pairs for a document type.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Self-Correcting Extraction Agent with Memory"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, halt = red).
- **App UI** — form to submit a document, live list of jobs with status pills, click-to-expand per-attempt timeline showing each field map, the scorer's verdict, the correction notes, and the confidence score.

Browser title: `<title>Akka Sample: Self-Correcting Extraction Agent with Memory</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every cycle's scorer verdict is recorded as an `EvalRecorded` event with `{ attemptNumber, decision, confidence, memoryHit }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/jobs/{id}`.
- **HT1 — halt** (`budget-cap`): when the loop reaches `budgetCap` without a `PASS`, the workflow ends with `JobBudgetExhausted`. The entity preserves every attempt, every correction note, the highest-confidence extraction, and a structured exhaustion reason. The system never deletes attempts or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `ExtractionAgent` → `prompts/extraction-agent.md`. Extracts structured fields from a raw document; on a correction call, takes the prior `CorrectionNotes` and the window-buffer context (last three attempts) as input and produces a revised `FieldMap`.
- `ScorerAgent` → `prompts/scorer-agent.md`. Grades a `FieldMap` against memory-grounded ground truth queried from `MemoryEntity`; returns `PASS` with a confidence score or `CORRECT` with up to four short correction bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a document; job progresses `EXTRACTING` → `SCORING` → `VERIFIED` within the budget cap; the App UI shows every attempt's field map and scorer verdict.
2. **J2 — budget exhausted** — Submit a document whose fields the scorer is forced to reject (test mode); job progresses through every attempt and lands in `BUDGET_EXHAUSTED` with the highest-confidence extraction preserved.
3. **J3 — memory recall** — After a job completes as `VERIFIED`, submit a second document of the same type; the Scorer's ground-truth context includes the confirmed field values from the first job.
4. **J4 — eval-event timeline** — The expanded view of any job shows one `EvalRecorded` event per scored attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-correcting-extraction demonstrating the evaluator-optimizer ×
ops-automation cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-ops-automation-self-correcting-extraction.
Java package io.akka.samples.selfcorrectingextractionagentwithmemory. Akka 3.6.0. HTTP port 9796.

Components to wire (exactly):
- 2 AutonomousAgents:
  * ExtractionAgent — definition() with
    capability(TaskAcceptance.of(EXTRACT).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(CORRECT_EXTRACTION).maxIterationsPerTask(3)).
    System prompt loaded from prompts/extraction-agent.md. Returns FieldMap{fields,
    confidence, extractedAt} for both EXTRACT and CORRECT_EXTRACTION. The
    CORRECT_EXTRACTION task takes (documentType, rawText, priorAttempts, CorrectionNotes)
    as inputs where priorAttempts is the window buffer (last 3 FieldMap results).
  * ScorerAgent — definition() with
    capability(TaskAcceptance.of(SCORE).maxIterationsPerTask(2)). System
    prompt from prompts/scorer-agent.md. Returns ScorerVerdict{decision, notes, confidence,
    scoredAt} where decision is the ScorerDecision enum (PASS | CORRECT) and confidence
    is a 0.0–1.0 double.

- 1 Workflow ExtractionWorkflow with steps:
    startStep -> extractStep -> scoreStep ->
    [decision PASS? verifyStep : (attemptCount < budgetCap ?
       correctStep -> extractStep : exhaustStep)] -> END.
  extractStep calls forAutonomousAgent(ExtractionAgent.class, jobId).runSingleTask(
    EXTRACT or CORRECT_EXTRACTION) then forTask(taskId).result(EXTRACT or
    CORRECT_EXTRACTION). scoreStep calls forAutonomousAgent(ScorerAgent.class,
    jobId).runSingleTask(SCORE) passing the latest FieldMap plus a memory context
    fetched via a direct call to MemoryEntity.getMemory(documentType).
  verifyStep emits JobVerified and writes confirmed field values to MemoryEntity.
  exhaustStep emits JobBudgetExhausted with the highest-confidence FieldMap
    as best-of and a structured exhaustionReason ("budget cap reached" plus the count).
  Override settings() with stepTimeout(60s) on extractStep and scoreStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(exhaustStep)).
  The workflow state carries a windowBuffer (List<FieldMap> of the last 3 attempts)
    updated after each extractStep; this buffer is passed to CORRECT_EXTRACTION.

- 2 EventSourcedEntities:
  * ExtractionJobEntity holding state ExtractionJob{jobId, documentType, rawText,
    budgetCap, JobStatus status, List<ExtractionAttempt> attempts,
    Optional<Integer> verifiedAttemptNumber, Optional<FieldMap> verifiedFieldMap,
    Optional<String> exhaustionReason, Instant createdAt, Optional<Instant> finishedAt}.
    JobStatus enum: EXTRACTING, SCORING, VERIFIED, BUDGET_EXHAUSTED.
    Events: JobCreated, AttemptExtracted, AttemptScored, JobVerified,
    JobBudgetExhausted, EvalRecorded.
    Commands: createJob, recordExtraction, recordVerdict, verify, exhaust,
    recordEval, getJob. emptyState() returns ExtractionJob.initial("", "", "", 4).
  * MemoryEntity keyed by documentType, holding a Map<String, MemoryRecord> of
    confirmed field values. Events: MemoryRecordWritten{documentType, fieldName,
    confirmedValue, confirmedAt}. Commands: writeMemory, getMemory.

- 1 EventSourcedEntity DocumentQueue with command submitDocument(documentType,
  rawText, submittedBy) emitting DocumentSubmitted{jobId, documentType, rawText,
  submittedBy, submittedAt}.

- 1 View JobsView with row type JobRow (mirrors ExtractionJob; the attempts list
  is bounded at budgetCap). Table updater consumes ExtractionJobEntity events.
  ONE query getAllJobs SELECT * AS jobs FROM jobs_view. No WHERE status filter —
  caller filters client-side (Lesson 2).

- 1 Consumer DocumentConsumer subscribed to DocumentQueue events; on
  DocumentSubmitted starts an ExtractionWorkflow with the jobId as the workflow id.

- 2 TimedActions:
  * DocumentSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-documents.jsonl and calls
    DocumentQueue.submitDocument.
  * EvalSampler — every 30s, queries JobsView.getAllJobs, finds attempts
    that have been scored but do not yet have a matching EvalRecorded event, and
    calls ExtractionJobEntity.recordEval(attemptNumber, decision, confidence,
    memoryHit). Idempotent per (jobId, attemptNumber).

- 2 HttpEndpoints:
  * ExtractionEndpoint at /api with POST /jobs, GET /jobs, GET /jobs/{id},
    GET /jobs/sse, GET /memory/{documentType}, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
    The POST /jobs body is {documentType, rawText, submittedBy?}; missing
    submittedBy defaults to "anonymous". documentType must be non-blank; rawText
    must be between 10 and 10000 chars; otherwise 400.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- ExtractionTasks.java declaring three Task<R> constants: EXTRACT (resultConformsTo
  FieldMap), CORRECT_EXTRACTION (FieldMap), SCORE (ScorerVerdict).
- Domain records FieldMap, CorrectionNotes, ScorerVerdict, ExtractionAttempt,
  ExtractionJob, MemoryRecord, DocumentSubmission; enums JobStatus, ScorerDecision.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9796 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  self-correcting-extraction.extraction.budget-cap = 4 and
  self-correcting-extraction.extraction.window-buffer-size = 3, overridable by
  env var.
- src/main/resources/sample-events/sample-documents.jsonl with 8 canned document
  lines, each shaped {"documentType":"invoice","rawText":"..."}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-event on-decision-eval,
  HT1 halt budget-cap) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = document-field-extraction,
  decisions.authority_level = draft-only, data.data_classes.pii = true (invoices
  may contain vendor PII), capabilities.* = false except content-generation = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/extraction-agent.md, prompts/scorer-agent.md loaded at agent startup as
  system prompts.
- README.md at the project root: title
  "Akka Sample: Self-Correcting Extraction Agent with Memory",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live job list with status pills, click-to-expand
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Self-Correcting Extraction Agent with Memory</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: extraction-agent.json, scorer-agent.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    extraction-agent.json — 6 FieldMap entries. Three are well-formed
      extractions for invoice documents (invoiceNumber, invoiceDate,
      vendorName, totalAmount all present and plausible). Two are
      partial extractions with one or two fields missing or malformed
      (used to exercise the CORRECT path). One has a deliberately wrong
      totalAmount format (letters instead of a numeric string) to
      exercise the scorer's correction notes.
    scorer-agent.json — 6 ScorerVerdict entries. Three return decision=PASS
      with confidence ≥ 0.90 and an empty CorrectionNotes. Three return
      decision=CORRECT with confidence 0.40–0.65 and a CorrectionNotes
      payload of two to four bullets (e.g., "invoiceDate should be
      ISO-8601 not MM/DD/YYYY", "totalAmount must be numeric; got 'N/A'",
      "vendorName is blank but present in memory as 'Acme Corp'").
- A MockModelProvider.seedFor(jobId, attemptNumber) helper makes the
  selection deterministic per (jobId, attemptNumber) so the same job in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. ExtractionAgent
  and ScorerAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an ExtractionTasks companion declaring the three Task<R>
  constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the ExtractionJob record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: ExtractionTasks.java is mandatory; generating ExtractionAgent or
  ScorerAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9796, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
