# Architecture — dual-llm-pdf-extract

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ExtractionEndpoint` is the entry point. It writes a `DocumentReceived` event to `DocumentQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `ExtractionWorkflow` per submission. The workflow redacts the document text, then calls three agents — `ClaudeExtractor` and `GeminiExtractor` in parallel over the same redacted text, then `ExtractionReconciler` to merge the two independent results. Each transition emits an event on `ExtractionEntity`. `ExtractionView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `PdfSimulator` drips sample document texts for demonstration, and `AgreementSampler` ticks every 5 minutes to grade one reconciled extraction for cross-model agreement, calling `EvalJudge`.

The PII sanitizer is not a component box of its own — it is the deterministic `PiiSanitizer` helper invoked inside `ExtractionWorkflow.sanitizeStep`. It makes no Akka call; it transforms the raw text into a redacted text and a count before the text is ever persisted.

## Interaction sequence

The sequence diagram traces the happy path (J1). Two notes matter. First, the sanitize step runs before any agent is called and before any text is persisted — the raw text lives only in the workflow's transient start command. Second, the `par` block: the two extractors run **concurrently** over the same redacted text, not in a chain. The workflow awaits both via a CompletionStage zip with a 60-second per-step timeout, which is the heart of the debate-multi-perspective pattern — both perspectives must be gathered before reconciliation begins.

## State machine

The extraction has four states. `INTAKE` is the initial state when `ExtractionCreated` fires. `EXTRACTING` is reached once the document is sanitized and both extractors are dispatched. The extraction then lands in one of two terminal states: `RECONCILED` (both extractors returned and the reconciler merged the results) or `DEGRADED` (an extractor timed out; reconciliation ran on partial input). An `AgreementScored` event may land on a `RECONCILED` extraction without changing its status.

## Entity model

`ExtractionEntity` is the system's source of truth; every transition writes one of seven event types. `DocumentQueue` is the audit log of submissions. `ExtractionView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for each extractor call and for the reconcile call. The default 5-second workflow step timeout is far too short for an LLM call (Lesson 4).
- Degraded path: an extractor timeout transitions to reconciliation-from-partial rather than failing the whole workflow; `failureReason` names the missing extractor.
- Idempotency: `(filename, submittedBy)` over a 10 s window deduplicates `POST /api/extractions`.
- View indexing: `getAllExtractions` has no `WHERE status` clause — the `ExtractionStatus` enum cannot be auto-indexed; callers filter client-side.
- Eval sampler: one extraction per 5-minute tick; the oldest `RECONCILED` extraction without an `agreementScore` wins.
- Sanitizer ordering: redaction happens before both extractors are dispatched and before the raw text could be persisted, which is what realises control S1.
- Dual-model configuration: `ClaudeExtractor` is wired to the `anthropic` model-provider block; `GeminiExtractor` is wired to the `googleai-gemini` block. If only one key is available at runtime, both extractors fall back to that provider — the cross-model comparison signal degrades but the extraction pipeline continues.
