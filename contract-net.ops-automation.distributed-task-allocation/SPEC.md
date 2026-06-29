# SPEC — Contract-Net Task Auctioneer

## 1. Overview

The Contract-Net Task Auctioneer implements the classic Contract-Net Protocol (CNP) as a durable, observable Akka service. A manager component announces work as structured tasks with requirements and a bidding deadline. Registered contractor agents submit bids expressing their capability and cost estimate. When the bidding window closes, an LLM-backed evaluator ranks bids against the task specification; the winning bid passes through a guardrail before the award is committed. The winning contractor executes the task. On completion, a second LLM agent scores the execution quality and updates the contractor's persistent performance record, throttling future awards if the score falls below threshold. On failure, a renegotiation workflow reopens bidding with the failed contractor excluded.

The sample is self-contained: it ships a task runner, a pool of simulated contractor agents, and a web UI for observing every protocol step in real time.

## 2. Goals & Non-Goals

**Goals**
- Demonstrate full CNP lifecycle: announce → bid → validate (guardrail) → award → execute → score (eval-event) → [renegotiate]
- Show two inline governance controls: before-bid-acceptance guardrail and on-completion eval-event
- Persist task and contractor state durably across restarts
- Stream live auction and leaderboard state to the UI via SSE
- Provide a renegotiation path that does not require manual intervention

**Non-Goals**
- Real contractor agents executing real work (simulated by the task runner)
- Multi-manager federation or cross-cluster task routing
- Bidding strategy optimization or auction-theory extensions
- SLA enforcement beyond the renegotiation trigger
- Production auth/authz (out of scope for a sample)

## 3. Coordination Pattern

**Pattern:** Contract-Net Protocol (CNP)

CNP separates task allocation into four roles:
- **Manager** — holds task ownership, controls the auction lifecycle, commits the award
- **Contractor pool** — registered agents that receive announcements and respond with bids
- **Evaluator** — LLM agent that ranks bids; invoked once per auction close
- **Scorer** — LLM agent that grades execution; invoked once per task completion or failure

The protocol steps map directly to entity command handlers and workflow steps:

```
Manager announces task
  → TaskAnnouncementEntity emits TaskAnnounced
  → AuctionWorkflow opens bidding window
  → Contractors submit bids → BidSubmitted events
  → Window closes → BidEvaluatorAgent ranks bids
  → Guardrail validates winning bid against task spec → BidValidated
  → AuctionWorkflow commits award → TaskAwarded
  → Contractor executes → TaskCompleted | TaskFailed
  → PerformanceScorerAgent grades outcome → PerformanceScored
  → [On failure] RenegotiationWorkflow reopens bidding
```

The contract between manager and contractor is enforced by the guardrail. The performance feedback loop is enforced by the eval-event control. Both controls are observable in the audit log.

## 4. Components

| Component | Type | Responsibility |
|-----------|------|----------------|
| `TaskAnnouncementEntity` | Event Sourced Entity | Owns task lifecycle state; processes commands AnnounceTask, SubmitBid, AwardTask, ReportCompletion, ReportFailure |
| `ContractorRegistryEntity` | Event Sourced Entity | Owns contractor registration and mutable performance score; processes RegisterContractor, UpdateScore, ThrottleContractor |
| `AuctionWorkflow` | Workflow | Orchestrates bidding window timer, bid collection, evaluator call, guardrail step, award commit, and completion monitoring |
| `RenegotiationWorkflow` | Workflow | Triggered by TaskFailed; excludes failed contractor, reopens bidding, assigns renegotiation attempt counter |
| `BidEvaluatorAgent` | AI Agent (LLM) | Receives task spec + bid list; returns ranked bid list with evaluation rationale per bid |
| `PerformanceScorerAgent` | AI Agent (LLM) | Receives task spec + completion report; returns numeric score (0–100) and textual feedback |
| `ActiveAuctionsView` | Key-Value View | Projects open TaskAnnouncementEntity states; exposes list of auctions with bid counts for SSE |
| `ContractorLeaderboardView` | Key-Value View | Projects ContractorRegistryEntity states; exposes ranked list sorted by performance score |
| `TaskAuctioneerEndpoint` | HTTP Endpoint | Routes all inbound REST and SSE requests; calls entity/workflow command methods |

## 5. Data Model

### TaskSpec
| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `title` | String | No | Short task title |
| `description` | String | No | Full requirements text passed to evaluator |
| `requiredCapabilities` | List\<String\> | No | Tags that contractors must declare |
| `maxBudget` | BigDecimal | No | Upper bound for acceptable bids |
| `biddingDeadlineSeconds` | int | No | Seconds from announcement before window closes |

### Bid
| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `bidId` | String | No | UUID |
| `contractorId` | String | No | References ContractorRegistryEntity |
| `costEstimate` | BigDecimal | No | Contractor's quoted price |
| `capacityNote` | String | Yes | Optional free-text on bandwidth/timeline |
| `submittedAt` | Instant | No | Server-assigned timestamp |

### ContractorProfile
| Field | Type | Nullable | Description |
|-------|------|----------|-------------|
| `contractorId` | String | No | UUID |
| `name` | String | No | Display name |
| `capabilities` | List\<String\> | No | Declared capability tags |
| `performanceScore` | int | No | 0–100; updated by PerformanceScorerAgent |
| `throttled` | boolean | No | True when score falls below threshold |
| `completedTasks` | int | No | Cumulative count |

### Task status enum
`ANNOUNCED` → `BIDDING` → `AWARDED` → `IN_PROGRESS` → `COMPLETED`
                                                      ↘ `FAILED` → `RENEGOTIATING` → (back to BIDDING)

### Events
| Event | Trigger |
|-------|---------|
| `TaskAnnounced` | AnnounceTask command accepted |
| `BidSubmitted` | SubmitBid command accepted |
| `BidValidated` | Guardrail passes winning bid |
| `TaskAwarded` | AwardTask command committed |
| `TaskStarted` | Contractor signals execution start |
| `TaskCompleted` | Contractor reports successful completion |
| `TaskFailed` | Contractor reports failure |
| `PerformanceScored` | PerformanceScorerAgent returns score |
| `ContractorThrottled` | Score drops below throttle threshold |
| `RenegotiationOpened` | RenegotiationWorkflow starts |

## 6. API Contract

| Method | Path | Summary |
|--------|------|---------|
| POST | `/tasks` | Announce a new task |
| GET | `/tasks/{taskId}` | Get task state |
| POST | `/tasks/{taskId}/bids` | Submit a bid |
| GET | `/tasks/{taskId}/bids` | List bids for a task |
| GET | `/auctions/active` | SSE stream of active auctions |
| POST | `/contractors/register` | Register a contractor agent |
| GET | `/contractors/{contractorId}` | Get contractor profile |
| GET | `/contractors/leaderboard` | SSE stream of contractor leaderboard |

See `reference/api-contract.md` for full request/response schemas and SSE event formats.

## 7. Agent Roles

### BidEvaluatorAgent
Invoked by `AuctionWorkflow` after the bidding window closes. Receives the full task spec and all submitted bids. Returns a ranked list of bids with a brief rationale for each ranking position. The workflow extracts the top-ranked bid and passes it to the guardrail step.

**Model requirements:** Instruction-following; context window sufficient for task spec (~500 tokens) plus up to 50 bids (~2000 tokens). No tool use required.

### PerformanceScorerAgent
Invoked by `AuctionWorkflow` after `TaskCompleted` or `TaskFailed` is recorded. Receives the original task spec and the contractor's completion report (free text). Returns a JSON object with `score` (0–100 integer) and `feedback` (string). The workflow writes this back to `ContractorRegistryEntity` via an `UpdateScore` command.

**Model requirements:** Instruction-following with structured output (JSON mode or prompted schema). No tool use required.

## 8. Control Mechanisms

### Control 1 — Before-Bid-Acceptance Guardrail (G-001)

**Mechanism:** `guardrail`, hook: `before-bid-acceptance`

After `BidEvaluatorAgent` returns the ranked bid list, `AuctionWorkflow` executes a synchronous validation step before committing the award. The validation checks that:
1. The winning contractor is registered and not throttled.
2. The bid's `costEstimate` does not exceed `TaskSpec.maxBudget`.
3. The contractor's declared capabilities cover all `TaskSpec.requiredCapabilities`.

If any check fails, the next-ranked bid is promoted and re-validated. If no bid passes, the task enters `RENEGOTIATING`. The guardrail outcome is recorded as a `BidValidated` event with an `accepted` or `rejected` payload, producing an auditable trace of every validation decision.

### Control 2 — On-Completion Eval-Event (E-001)

**Mechanism:** `eval-event`, hook: `on-completion-eval`

When `TaskCompleted` or `TaskFailed` is written to the task entity, `AuctionWorkflow` triggers `PerformanceScorerAgent`. The returned score is persisted via `PerformanceScored` on `ContractorRegistryEntity`. If the cumulative rolling score (last 5 tasks) drops below 40, a `ContractorThrottled` event is emitted and the contractor's `throttled` flag is set to `true`. Throttled contractors are filtered out of future bid acceptance by the guardrail (Control 1), closing the feedback loop without manual intervention.

## 9. Constraints & Lessons

- **Lesson 1** — Each entity (`TaskAnnouncementEntity`, `ContractorRegistryEntity`) is the single source of truth for its state; no cross-entity reads inside command handlers.
- **Lesson 4** — `AuctionWorkflow` and `RenegotiationWorkflow` use durable step definitions; all LLM calls are wrapped in workflow steps so retries do not re-submit bids or double-award.
- **Lesson 6** — `BidEvaluatorAgent` and `PerformanceScorerAgent` are stateless; their inputs are passed entirely in each invocation; no session state is held between calls.
- **Lesson 7** — Bidding deadline is implemented as a workflow timer step, not a scheduled job or polling loop.
- **Lesson 8** — `ActiveAuctionsView` and `ContractorLeaderboardView` are read-side projections; the endpoint never queries entities directly for list responses.
- **Lesson 9** — SSE streams for `/auctions/active` and `/contractors/leaderboard` are backed by view subscriptions, not polling.
- **Lesson 10** — Events are immutable facts; `TaskFailed` is never corrected in place — `RenegotiationOpened` is a separate subsequent event.
- **Lesson 11** — `ContractorRegistryEntity` uses optimistic concurrency; the `UpdateScore` command carries the expected sequence number to avoid lost updates when two completions arrive concurrently.
- **Lesson 12** — `BidEvaluatorAgent` prompt includes the full task spec and all bids in a single context; no external retrieval at evaluation time.
- **Lesson 13** — The guardrail step in `AuctionWorkflow` is a pure function; it reads from inputs provided by the workflow, never from a live entity read inside the step.
- **Lesson 23** — All workflow step outputs are serialized to the Akka journal before proceeding; no in-memory-only workflow state.
- **Lesson 24** — State machine labels in PLAN.md diagrams use Mermaid CSS class overrides so state names render clearly against the dark theme.
- **Lesson 25** — The `PerformanceScorerAgent` output schema is validated before the `PerformanceScored` event is written; malformed LLM output causes the step to retry, not to write a corrupt event.
- **Lesson 26** — UI tab switching in `reference/ui-mockup.md` uses data-attribute selectors, not class toggling, for predictable CSS specificity.

## 10. Open Questions

1. **Bid window extension** — Should a task manager be able to extend the bidding deadline after announcement? Current design treats the deadline as immutable once `TaskAnnounced` is emitted.
2. **Partial bids** — Should a contractor be able to revise a bid before the window closes, or is the first bid final? Revision would require a `BidRevised` event and an updated evaluation trigger.
3. **Throttle recovery** — How does a throttled contractor regain eligibility? Current design requires a manual `UnthrottleContractor` command; an automatic time-based recovery could be added.
4. **Multi-task contractors** — Can a contractor win multiple tasks simultaneously? Current design does not enforce a capacity cap; the `capacityNote` field is advisory only.
5. **Evaluator fallback** — If `BidEvaluatorAgent` fails after retries, should the workflow fall back to lowest-cost selection or halt the auction?

## 11. Identity

| Field | Value |
|-------|-------|
| Folder | `contract-net.ops-automation.distributed-task-allocation` |
| Maven group | `io.akka.samples` |
| Artifact ID | `contract-net-ops-automation-distributed-task-allocation` |
| Java package | `io.akka.samples.contractnettaskauctioneer` |
| Akka version | `3.6.0` |
| HTTP port | `9202` |

## 12. Glossary

| Term | Definition |
|------|------------|
| **Contract-Net Protocol (CNP)** | A distributed task-allocation mechanism in which a manager broadcasts a task announcement, collects bids from a pool, and awards the task to the winning bidder. |
| **Task announcement** | The manager's broadcast of a task specification, including requirements, budget cap, and bidding deadline. |
| **Bid** | A contractor agent's response to an announcement, stating cost estimate and optionally a capacity note. |
| **Guardrail** | A synchronous validation step that runs before an award is committed; prevents invalid or throttled bids from being accepted. |
| **Eval-event** | An asynchronous evaluation triggered by a lifecycle event (task completion or failure); produces a score persisted to the contractor record. |
| **Throttle** | A flag on a contractor that bars them from winning new awards until manually cleared; set automatically when rolling performance score drops below threshold. |
| **Renegotiation** | The process of reopening bidding for a failed task, excluding the contractor that failed, managed by `RenegotiationWorkflow`. |

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
