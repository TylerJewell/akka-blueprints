# SPEC — hitl-pdf-extract

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL PDF Data Extraction.
**One-line pitch:** A user submits a PDF document URL; `ExtractionAgent` extracts structured fields; `RedactionAgent` scrubs PII before display; the workflow pauses at a human review gate for low-confidence results; on approval the validated data is posted downstream.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the ops-automation domain: a 4-task graph that extracts, redacts PII, conditionally pauses at a human review gate, and posts only when data is approved. The governance pattern is an application-level human approval gate between extraction and downstream posting, a PII sanitizer that runs before any extracted field is displayed to a reviewer, and a tool-permission guardrail that blocks the post step unless the document is approved.

## 3. User-facing flows

1. A client POSTs a document URL to `/api/extraction-request`. The response returns `{ documentId }`. The document appears in the UI in `EXTRACTED` once `ExtractionAgent` finishes (typically 5–30 s), with the extracted fields and an overall confidence score visible.
2. If confidence is at or above the configured threshold, the workflow advances automatically to `APPROVED` and then `POSTED`. No human action required.
3. If confidence is below the threshold, the document enters `PENDING_REVIEW`. The reviewer reads the fields and clicks Approve. This POSTs to `/api/documents/{documentId}/approve`. The workflow resumes, data is posted downstream, and status becomes `POSTED`.
4. The reviewer clicks Reject with a reason. This POSTs to `/api/documents/{documentId}/reject`. The document moves to terminal `REJECTED` and the reason is shown. The post step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ExtractionAgent | AutonomousAgent | Extracts structured fields from a PDF; returns `ExtractionResult{fields, confidence}` | ExtractionWorkflow | DocumentEntity |
| RedactionAgent | AutonomousAgent | Redacts PII from extracted fields before human review; returns `RedactedResult{fields}` | ExtractionWorkflow | DocumentEntity |
| ExtractionWorkflow | Workflow | Orchestrates extract → redact → await review → post downstream | ExtractionEndpoint | ExtractionAgent, RedactionAgent, DocumentEntity |
| DocumentEntity | EventSourcedEntity | Holds the document state and lifecycle events | ExtractionWorkflow, ExtractionEndpoint | DocumentsView |
| DocumentsView | View | CQRS read model of all documents | DocumentEntity | ExtractionEndpoint |
| ExtractionEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | ExtractionWorkflow, DocumentEntity, DocumentsView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Document` (DocumentEntity state and DocumentsView row): `id` (String), `documentUrl` (`Optional<String>`), `status` (DocumentStatus enum), `confidence` (`Optional<Double>`), and lifecycle fields all `Optional<T>`: `submittedAt`, `extractedAt`, `rawFields`, `redactedFields`, `reviewedAt`, `reviewedBy`, `reviewerComment`, `rejectedAt`, `rejectedBy`, `rejectReason`, `postedAt`, `postTarget`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`DocumentStatus` enum: `EXTRACTED`, `PENDING_REVIEW`, `APPROVED`, `REJECTED`, `POSTED`.

Events: `DocumentExtracted`, `DocumentRedacted`, `DocumentPendingReview`, `DocumentApproved`, `DocumentRejected`, `DocumentPosted`.

Domain records: `ExtractionResult(Map<String,String> fields, double confidence)`, `RedactedResult(Map<String,String> fields)`, `ReviewDecision(String reviewedBy, String comment)`, `PostReceipt(String postTarget, String postedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/extraction-request                  -> { documentId }
POST /api/documents/{documentId}/approve      -> 200 | 404
POST /api/documents/{documentId}/reject       -> 200 | 404
GET  /api/documents                           -> { documents: [Document, ...] }
GET  /api/documents/{documentId}              -> Document
GET  /api/documents/sse                       -> Server-Sent Events of Document
GET  /api/metadata/eval-matrix                -> text/yaml
GET  /api/metadata/risk-survey                -> text/yaml
GET  /api/metadata/readme                     -> text/markdown
GET  /                                        -> 302 /app/index.html
GET  /app/{*path}                             -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL PDF Data Extraction</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits a document URL, lists documents live via SSE, and shows Approve/Reject buttons on `PENDING_REVIEW` documents that have redacted fields. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `ExtractionWorkflow` pauses at the await-review task when confidence is below threshold; `/api/documents/{id}/approve` and `/api/documents/{id}/reject` resume it.
- **S1 — sanitizer · pii.** `RedactionAgent` runs before extracted fields are persisted for display; sensitive tokens (names, IDs, account numbers) are replaced with `[REDACTED]` markers in `redactedFields`.
- **G1 — guardrail · before-tool-call.** A guardrail on the downstream post step verifies `DocumentEntity.status == APPROVED` before the simulated post tool runs.

## 9. Agent prompts

- `ExtractionAgent` — extracts structured fields from a document. See `prompts/extraction-agent.md`.
- `RedactionAgent` — redacts PII from extracted field values. See `prompts/redaction-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Extract a document.** POST a URL; within ~30 s a document appears in `EXTRACTED` with non-empty `rawFields`.
2. **High-confidence auto-post.** Submit a document whose extracted confidence meets or exceeds 0.85; it transitions automatically through `APPROVED` → `POSTED` without human action.
3. **Low-confidence review and approve.** Submit a document below the confidence threshold; it enters `PENDING_REVIEW`; a reviewer approves it; it reaches `POSTED` with a non-null `postTarget`.
4. **Reject a low-confidence extraction.** Reject a `PENDING_REVIEW` document with a reason; it moves to terminal `REJECTED` and the reason shows.
5. **Post guard.** The post step is never reached for a document that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named hitl-pdf-extract demonstrating the human-in-loop-gate ×
ops-automation cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-ops-automation-hitl-pdf-extract.
Java package io.akka.samples.extractdatafrompdfswithhumanintheloopvalidationcradlai.
Akka 3.6.0. HTTP port 9828.

Components to wire (exactly):
- 2 AutonomousAgents: ExtractionAgent (extracts structured fields from a PDF
  document, returns a typed ExtractionResult{fields, confidence}) and
  RedactionAgent (redacts PII from extracted fields, returns a typed
  RedactedResult{fields}). Each declares definition() returning an AgentDefinition
  with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow ExtractionWorkflow with four tasks: extractStep -> redactStep ->
  awaitReviewStep -> postStep. extractStep calls
  forAutonomousAgent(ExtractionAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...), writes recordExtraction on DocumentEntity.
  redactStep calls RedactionAgent and writes recordRedaction on DocumentEntity.
  awaitReviewStep reads DocumentEntity.getDocument; if confidence >= 0.85 it
  auto-approves by calling approve on DocumentEntity; if PENDING_REVIEW it
  self-schedules a 5-second resume timer; on APPROVED it transitions to postStep;
  on REJECTED it ends. postStep calls the simulated post tool and writes
  recordPost on DocumentEntity. Override settings() with stepTimeout(60s) on
  extractStep, redactStep, and postStep; WorkflowSettings is nested in Workflow
  (no import).
- 1 EventSourcedEntity DocumentEntity holding a Document record with id,
  documentUrl (Optional<String>), DocumentStatus enum
  {EXTRACTED,PENDING_REVIEW,APPROVED,REJECTED,POSTED}, confidence
  (Optional<Double>), and Optional lifecycle fields (submittedAt, extractedAt,
  rawFields, redactedFields, reviewedAt, reviewedBy, reviewerComment, rejectedAt,
  rejectedBy, rejectReason, postedAt, postTarget). Events: DocumentExtracted,
  DocumentRedacted, DocumentPendingReview, DocumentApproved, DocumentRejected,
  DocumentPosted. Commands: recordExtraction, recordRedaction, setPendingReview,
  approve, reject, recordPost, getDocument. emptyState() returns
  Document.initial("") with no commandContext() reference (Lesson 3).
- 1 View DocumentsView with row type Document, table updater consuming
  DocumentEntity events. ONE query: getAllDocuments SELECT * AS documents FROM
  documents_view. No WHERE status filter (Akka cannot auto-index enum columns,
  Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: ExtractionEndpoint at /api with extraction-request (starts an
  ExtractionWorkflow with a fresh UUID), approve, reject, documents list (filter
  client-side from getAllDocuments), single document, SSE stream, and three
  /api/metadata/* endpoints serving the YAML/MD files from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- ExtractionTasks.java declaring two Task<R> constants: EXTRACT (resultConformsTo
  ExtractionResult) and REDACT (resultConformsTo RedactedResult).
- ExtractionResult(Map<String,String> fields, double confidence),
  RedactedResult(Map<String,String> fields), ReviewDecision(String reviewedBy,
  String comment), PostReceipt(String postTarget, String postedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9828
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 3 controls (H1, S1, G1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ExtractionAgent -> ExtractionResult, RedactionAgent -> RedactedResult; see
  src/main/resources/mock-responses/{extraction-agent,redaction-agent}.json with
  4–6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- extraction-agent.json: 4–6 entries, each { "fields": { "invoiceNumber": "...",
  "vendor": "...", "amount": "...", "date": "...", "lineItems": "..." },
  "confidence": 0.65–0.95 (vary so some trigger HITL) }.
- redaction-agent.json: 4–6 entries, each { "fields": { "invoiceNumber": "...",
  "vendor": "[REDACTED]", "amount": "...", "date": "...", "lineItems": "..." } }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  ExtractionTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9828 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching matches by data-tab/data-panel attribute, never
  NodeList index; no hidden zombie panels in the DOM.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
