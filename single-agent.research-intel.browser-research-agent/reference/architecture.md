# Architecture — reddit-search

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ResearchEndpoint` accepts a job submission, calls `ResearchJobEntity.enqueue`, then starts a `ResearchWorkflow` instance. The workflow's `browseStep` calls `BrowserResearchAgent` — the single AutonomousAgent — with the research topic as task instructions. The agent issues `navigate(url)` tool calls to drive a headless browser; before each navigation executes, `NavigationGuardrail` validates the destination and either allows or blocks it. Simultaneously, after each `PageVisited` event lands on the entity, `BudgetEnforcer` checks whether the session has reached its page ceiling. When the agent finishes or the budget fires, the workflow's `scoreStep` runs `RelevanceScorer` to re-rank the collected posts, then writes `ReportRecorded` to the entity. `ResearchView` projects every entity event into a read-model row; `ResearchEndpoint` serves that read model over REST and SSE.

The graph deliberately has no second agent. `BudgetEnforcer` and `RelevanceScorer` are deterministic supporting classes — they do not call a model. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Two governance mechanisms are active during the session:

1. **Navigation guardrail** — fires before every `navigate(url)` call. In the happy path all URLs are on Reddit and HTTPS, so all calls return `allow`. When a blocked URL appears (J2), the guardrail returns a structured rejection immediately and the agent proposes an alternative.
2. **Budget enforcer** — polled after every `PageVisited` event. In the happy path the session ends before the ceiling. In J3 the ceiling fires and the workflow transitions to `BUDGET_EXHAUSTED` carrying the partial report.

The `browseStep` timeout is 180 s — long enough to accommodate multiple sequential page visits plus LLM round-trips, but bounded so a hung browser tool cannot block indefinitely.

## State machine

Five states. The interesting paths:

- The happy path is `QUEUED → BROWSING → REPORT_READY`.
- The budget path is `BROWSING → BUDGET_EXHAUSTED` — a partial report is preserved and the job is considered complete, not failed.
- Two failure transitions land in `FAILED`: an unrecoverable agent error during `BROWSING`, and a workflow-start error during `QUEUED`.
- There is no `APPROVED` state. The report is informational; the researcher reads it and acts outside the system.

## Entity model

`ResearchJobEntity` is the source of truth. It emits six event types. `ResearchView` projects every event into a row used by the UI. `ResearchWorkflow` both reads (`getJob`) and writes (`markBrowsing`, `recordPageVisit`, `recordReport`, `recordBudgetExhausted`, `fail`) on the entity. The relationship between `BrowserResearchAgent` and `ResearchReport` is "returns" — the agent's task result is the report record.

## Defence-in-depth governance flow

For any report that lands in the entity log, the session passed through:

1. **NavigationGuardrail** — every URL the agent visited was validated before the browser executed it; the audit log holds the full sequence of visited URLs via `PageVisited` events.
2. **BrowserResearchAgent** — one model call, one structured output per session.
3. **BudgetEnforcer** — the agent could not exceed the page ceiling declared by the user; cost is bounded at submission time.
4. **RelevanceScorer** — the final post ranking is deterministic and auditable; the scorer's weights are documented in Javadoc and produce the same output given the same input.

Each control is independent. The guardrail does not know about the budget, and the budget enforcer does not know about the guardrail's allow-list.
