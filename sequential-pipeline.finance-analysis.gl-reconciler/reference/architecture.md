# Architecture — gl-reconciler

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `ReconciliationEndpoint` accepts an `{accountSetId}` POST, writes `RunCreated` onto `LedgerReconciliationEntity`, and starts `ReconciliationWorkflow` keyed by `"recon-" + runId`. The workflow's first step (`fetchStep`) emits `FetchStarted`, then calls `LedgerAgent` with `TaskDef.taskType(FETCH_ENTRIES)` and a `phase = FETCH` metadata tag. The agent invokes `FetchTools.fetchLedgerEntries` and `FetchTools.fetchAccountBalances`; every task result passes through `PostingGuardrail` before the workflow accepts it. Once the agent returns a `LedgerSnapshot`, the workflow writes `EntriesFetched` onto the entity and advances to `reconcileStep`. After `ReconciliationCompleted` lands, `MaterialVarianceChecker` runs synchronously. If any variance is material, the workflow writes `EscalationRaised`, transitions to `PENDING_APPROVAL`, and waits for a controller command. On approval, or when no escalation is needed, `draftStep` runs with `phase = DRAFT`. The `PostingGuardrail` fires on the returned `JournalEntry` — enforcing balance and valid account codes — before the workflow writes `ValidationPassed` and `JournalPosted`. `ReconciliationView` projects every event into a read-model row; `ReconciliationEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `MaterialVarianceChecker` and `PostingGuardrail` are deterministic; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1) — no material variance, balanced first draft. Two properties are worth pausing on:

1. The task boundary IS the dependency contract. Between `fetchStep` and `reconcileStep`, the workflow writes `EntriesFetched` onto the entity. The next step reads `snapshot` from the entity to build the RECONCILE task's instruction context. The agent never sees FETCH-phase data inside the RECONCILE task's conversation; the typed handoff is the only path information travels.
2. The `before-agent-response` guardrail fires at the task-result boundary, not at individual tool calls. This means the guardrail sees the complete `JournalEntry` and can enforce multi-line balance — a property that cannot be checked tool-call by tool-call.

Agent calls are bounded by per-step timeouts (60 s per agent-calling step). `hitlStep` has a 72 h timeout to give the controller a full business day. `validateStep` is essentially synchronous.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → FETCHING → FETCHED → RECONCILING → RECONCILED → DRAFTING → DRAFT_VALIDATED → POSTED`.
- When a material variance is detected, the run forks: `RECONCILED → PENDING_APPROVAL → DRAFTING → DRAFT_VALIDATED → POSTED` (on approval) or `RECONCILED → PENDING_APPROVAL → REJECTED` (on rejection).
- Three failure transitions land in `FAILED`: an agent error during `FETCHING`, `RECONCILING`, or `DRAFTING`. A failed run's prior data is preserved on the entity — the UI shows the partial state.
- `ValidationFailed` is a side-event recorded when the posting guardrail rejects a draft; it does not change the status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no asynchronous posted-to-ERP state. The blueprint stops at `POSTED` (the entry is accepted by the service) and leaves ERP integration as a deployer extension.

## Entity model

`LedgerReconciliationEntity` is the source of truth. It emits fourteen event types — two lifecycle starts per phase, the escalation lifecycle (raise / approve / reject), the validation events, the posting, the failure, and the initial creation. `ReconciliationView` projects every event into a row used by the UI. `ReconciliationWorkflow` both reads and writes on the entity across the full run lifecycle. The relationship between `LedgerAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload, after passing through `PostingGuardrail`.

## Defence-in-depth governance flow

For any run that reaches `POSTED`, the account set passed through:

1. **Posting-safety guardrail** — every agent task result is filtered. An unbalanced `JournalEntry` or one with an invalid account code is rejected before the workflow writes it; a `ValidationFailed` event records the finding for audit.
2. **LedgerAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
3. **Material variance escalation** — any run with a variance exceeding the configured threshold is paused at `PENDING_APPROVAL` until a named controller issues an explicit decision. The controller's identity and decision are recorded on the entity.

Each layer is independent. The posting guardrail does not check variance materiality; the escalation path does not re-validate the journal. Removing one layer opens an explicit gap the other does not silently cover.
