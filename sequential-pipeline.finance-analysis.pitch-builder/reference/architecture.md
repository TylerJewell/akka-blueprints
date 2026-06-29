# Architecture — pitch-builder

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `PitchbookEndpoint` accepts a `{target}` POST, writes `PitchbookCreated` onto `PitchbookEntity`, and starts `PitchbookPipelineWorkflow` keyed by `"pipeline-" + pitchbookId`. The workflow's first step (`researchStep`) emits `ResearchStarted`, then calls `PitchAgent` with `TaskDef.taskType(RESEARCH_TARGET)` and a `phase = RESEARCH` metadata tag. The agent invokes `ResearchTools.searchFilings` and `ResearchTools.fetchHeadline`. Once the agent returns a raw `ResearchPack`, the workflow passes it through `PiiSanitizer.sanitize()` — a synchronous in-process call — which redacts any personal-name, personal-email, and direct-phone patterns and appends `PiiRedactionLogged` events. The sanitized pack is written to the entity as `TargetResearched`, and the workflow advances to `comparablesStep`. The COMPARABLES task calls `ComparablesTools.selectPeers` and `ComparablesTools.buildMultiplesTable`; the result is written as `ComparablesBuilt`. Then `draftStep` runs the DRAFT task; after the agent produces a `Pitchbook`, the `CitationValidator` guardrail fires before the workflow writes `DraftWritten`. If the guardrail rejects the draft, the agent retries. After `DraftWritten` lands, `validationStep` runs `CitationValidator` as a final scoring pass — no LLM call — and writes `ValidationScored`. `PitchbookView` projects every event into a read-model row; `PitchbookEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `PiiSanitizer` and `CitationValidator` are deterministic classes; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The PII sanitizer runs at the task boundary. The raw `ResearchPack` that the agent returned from the RESEARCH task passes through `PiiSanitizer` before the workflow writes it onto the entity or uses it as the next task's instruction context. The agent's COMPARABLES and DRAFT tasks never see the unredacted text.
2. The task boundary IS the dependency contract. Between `researchStep` and `comparablesStep`, the workflow writes `TargetResearched` (with the sanitized pack). The next step reads `research` from the entity to build the COMPARABLES task's instruction context. The agent never sees research-phase context inside the comparables task's conversation; the typed handoff is the only path information travels.
3. The `CitationValidator` fires before each DRAFT task response is accepted. The guardrail reads the in-flight task's `phase` metadata to identify DRAFT_PITCHBOOK responses, then checks citations against the recorded `CompsTable`. A draft citing a hallucinated peer or out-of-range multiple is rejected before the workflow writes it to the entity.

Agent steps are bounded by per-step timeouts (60 s on research / comparables / draft). `validationStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → RESEARCHING → RESEARCHED → COMPS_RUNNING → COMPS_READY → DRAFTING → DRAFTED → VALIDATED`.
- Three failure transitions land in `FAILED`: an agent error during `RESEARCHING`, `COMPS_RUNNING`, or `DRAFTING`. A `FAILED` pitchbook's prior data is preserved on the entity — the UI shows the partial state.
- `PiiRedactionLogged` and `CitationRejected` are side-events recorded for audit; they do not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `SENT` or `APPROVED` state. The pitchbook is advisory; the analyst reviews it and sends it to the client outside the system. The blueprint deliberately stops at `VALIDATED`.

## Entity model

`PitchbookEntity` is the source of truth. It emits eleven event types — three lifecycle starts, three lifecycle completions, the validation score, the PII redaction audit record, the citation rejection audit record, the failure, and the initial creation. `PitchbookView` projects every event into a row used by the UI. `PitchbookPipelineWorkflow` both reads (`getPitchbook`) and writes (`startResearch`, `recordResearch`, `startComps`, `recordComps`, `startDraft`, `recordDraft`, `recordValidation`, `logPiiRedaction`, `logCitationRejection`, `fail`) on the entity. The relationship between `PitchAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload after any sanitization or validation step.

## Defence-in-depth governance flow

For any pitchbook that reaches the entity log, the target passed through:

1. **PII sanitizer** — every `RawResearchItem.metadata` field is scanned before the research pack is written to the entity or forwarded downstream. Personal contact details are replaced with `[REDACTED]` and logged with field-level precision.
2. **PitchAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Before-agent-response guardrail** — every `Pitchbook` draft is checked for citation provenance before it is accepted. A draft citing a peer or multiple not in the recorded `CompsTable` is rejected; the agent self-corrects inside its iteration budget.
4. **On-decision scoring** — after `DraftWritten` lands, the same `CitationValidator` logic runs as a final 1–5 scoring pass. The score and rationale are visible on the UI card; pitchbooks with score ≤ 2 are flagged for analyst scrutiny.

Each step is independent. The PII sanitizer does not check citations; the citation validator does not check for PII. Removing one of them opens an explicit gap the other does not silently cover.
