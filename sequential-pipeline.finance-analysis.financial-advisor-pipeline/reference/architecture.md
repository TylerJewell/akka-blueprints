# Architecture — financial-advisor-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs four tasks in sequence. `AdvisoryEndpoint` accepts a `{query}` POST, writes `AdvisoryCreated` onto `AdvisoryEntity`, and starts `AdvisorPipelineWorkflow` keyed by `"advisory-" + advisoryId`. The workflow's first step (`researchStep`) emits `ResearchStarted`, then calls `FinancialAdvisorAgent` with `TaskDef.taskType(RESEARCH_MARKET)` and a `phase = RESEARCH` metadata tag. The agent invokes `ResearchTools.fetchMarketData` and `ResearchTools.lookupBenchmark`. Before the response leaves the agent, `DisclaimerGuardrail` prepends the regulated disclaimer and records `DisclaimerInjected` on the entity; then `SectorSanitizer` scans for prohibited patterns and records `SanitizerFired` for any matches.

Once the sanitized `MarketSnapshot` returns, the workflow writes `MarketResearched` and advances to `strategyStep` — same pattern, DEFINE_STRATEGY task with `phase = STRATEGY`. Then `executionStep` runs PLAN_EXECUTION, and finally `riskStep` runs ASSESS_RISK. After the risk profile is recorded, the workflow assembles a `FinancialReport` from all four typed results, writes `ReportAssembled`, then runs `ComplianceScorer` and writes `ComplianceScored`. `AdvisoryView` projects every event into a read-model row; `AdvisoryEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `ComplianceScorer` is a deterministic rule-based scorer, and `DisclaimerGuardrail` and `SectorSanitizer` are pure text transformers — that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. **The task boundary is the dependency contract.** Between `researchStep` and `strategyStep`, the workflow writes `MarketResearched` onto the entity. The next step then reads `snapshot` from the entity to build the STRATEGY task's instruction context. The agent never sees research-phase context inside the strategy task's conversation; the typed handoff is the only path information travels.

2. **Governance fires on every outbound response, not once per pipeline.** `DisclaimerGuardrail` and `SectorSanitizer` run on each of the four phase responses — RESEARCH, STRATEGY, EXECUTE, ASSESS. The entity's `disclaimerLog` and `sanitizerLog` are therefore full per-pipeline records, not per-session records. A reader auditing the advisory can see exactly which response triggered a sanitizer redaction.

3. **The agent is stateless across phases.** The workflow passes each task's typed result as the next task's instruction context. The agent does not hold RESEARCH context when running the STRATEGY task. This is the sequential-pipeline invariant: each phase's LLM call is isolated to that phase's typed input.

Each agent call is bounded by a per-step timeout (70 s on all four task steps). `ComplianceScorer` and the governance hooks are synchronous and finish in milliseconds.

## State machine

Ten states. The interesting paths:

- The happy path walks `CREATED → RESEARCHING → RESEARCHED → STRATEGIZING → STRATEGIZED → PLANNING → PLANNED → ASSESSING → EVALUATED`.
- Four failure transitions land in `FAILED`: an agent error during any of the four task steps. A `FAILED` advisory's prior data is preserved on the entity — the UI shows the partial state up to the last successful phase.
- `DisclaimerInjected` and `SanitizerFired` are side-events recorded for audit; they do not transition status. The advisory continues regardless of redaction count.

There is no `APPROVED` or `SUBMITTED` state. The advisory is educational; the reader consumes it and acts through their own channels. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`AdvisoryEntity` is the source of truth. It emits fourteen event types — four lifecycle starts, four lifecycle completions, report assembly, compliance scoring, disclaimer injection, sanitizer firing, initial creation, and failure. `AdvisoryView` projects every event into a row used by the UI. `AdvisorPipelineWorkflow` both reads and writes on the entity across all four phases. The relationship between `FinancialAdvisorAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any advisory that lands in the entity log, every outbound response passed through:

1. **Disclaimer guardrail** — every agent response has the regulated educational-purpose disclaimer prepended before the result reaches the workflow or the client. A `DisclaimerInjected` audit event records the exact text and timestamp.
2. **Sector sanitizer** — every agent response is scanned for three prohibited pattern classes. Matches are redacted in-place; `SanitizerFired` events record each intervention. The advisory content that reaches the entity and the UI is the post-redaction text.
3. **FinancialAdvisorAgent (4 task runs)** — four model calls, four structured outputs. Each task's typed result is the dependency handoff to the next phase.
4. **On-decision compliance evaluator** — every emitted `FinancialReport` gets a 1–5 score. Risk-band presence, mitigation coverage, allocation breadth, and source traceability are each worth one point on a base of 1.

Each layer is independent. The disclaimer guardrail does not check for prohibited phrases; the sanitizer does not check for disclaimer presence; the compliance scorer does not re-scan text. Removing one layer opens an explicit gap the others do not cover.
