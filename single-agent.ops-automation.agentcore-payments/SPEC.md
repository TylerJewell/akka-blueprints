# SPEC: Agent-Initiated Payment Settlement

## 1. Overview

**System name:** Agent-Initiated Payment Settlement  
**One-line pitch:** A single AI agent autonomously pays external APIs within a per-task spend envelope, with human approval gates for above-threshold payments and automatic halting on budget breach.

This blueprint demonstrates three Akka governance mechanisms working together:

1. **HITL (application)** — payments above a configurable threshold pause execution and require explicit human approval before the payment executes.
2. **Halt (budget-cap)** — when accumulated spend for a task reaches its budget ceiling, the agent halts all further payment attempts immediately.

The sample is intentionally bounded: one task, one agent session, deterministic state transitions, and two independent enforcement points that can be tested in isolation.

---

## 2. Business context

Operations teams frequently provision autonomous agents to call paid external data APIs — firmographic enrichment, market-data feeds, geocoding services, and similar capabilities. Allowing unconstrained spending creates financial risk; requiring manual approval for every call eliminates the automation benefit.

This system provides a middle path: the agent works freely within a per-payment threshold, holds larger payments for review, and self-terminates when the task budget is consumed. Finance and operations teams get a governed, auditable record of every payment decision without being in the critical path for routine calls.

---

## 3. Goals and non-goals

### Goals

- Model a payment task as an event-sourced entity with full spend history.
- Enforce a per-payment approval threshold via a durable workflow.
- Enforce a per-task budget cap that halts the agent without human intervention.
- Provide a streaming read model for real-time spend visibility.
- Persist every payment decision, approval, rejection, and halt as an immutable event.

### Non-goals

- Actual money movement — the sample records payment intents; real settlement integration is a deployer concern.
- Multi-task budget aggregation across agents or accounts.
- Dynamic threshold adjustment after task creation.
- Authentication and authorization — deployers add auth at the HTTP gateway layer.

---

## 4. Component inventory

| Component | Akka primitive | Responsibility |
|---|---|---|
| `PaymentTaskEntity` | Event-sourced entity | Owns task state, accumulated spend, payment records, and budget cap enforcement |
| `PaymentApprovalWorkflow` | Workflow | Orchestrates the HITL approval loop: holds a payment, notifies reviewer, waits for decision, resumes or rejects |
| `PaymentAgentSession` | Autonomous agent | Reads task description and decides which payments to initiate; calls `PaymentTaskEntity` to record each intent |
| `PaymentLedgerView` | Key-value view | Projects current task spend, budget remaining, and per-payment status for read access |
| `PaymentTaskEndpoint` | HTTP endpoint | Exposes task submission, approval decisions, ledger reads, and SSE streaming |

---

## 5. Data model

### Java records

**`PaymentTaskRequest`**

| Field | Type | Nullable | Description |
|---|---|---|---|
| `taskId` | `String` | No | Caller-assigned task identifier |
| `description` | `String` | No | Natural-language description of what the agent should pay for |
| `budgetCents` | `long` | No | Maximum total spend for this task in cents |
| `approvalThresholdCents` | `long` | No | Payments at or above this amount require human approval |

**`PaymentIntent`**

| Field | Type | Nullable | Description |
|---|---|---|---|
| `paymentId` | `String` | No | Agent-assigned identifier for this specific payment |
| `targetApi` | `String` | No | External API endpoint or service identifier |
| `amountCents` | `long` | No | Requested payment amount in cents |
| `purpose` | `String` | No | Agent's rationale for this payment |
| `requestedAt` | `Instant` | No | Timestamp when the agent submitted the intent |

**`ApprovalDecision`**

| Field | Type | Nullable | Description |
|---|---|---|---|
| `paymentId` | `String` | No | Payment being decided |
| `decision` | `ApprovalStatus` | No | `APPROVE` or `REJECT` |
| `reviewer` | `String` | No | Identity of the human reviewer |
| `decidedAt` | `Instant` | No | Timestamp of the decision |
| `note` | `Optional<String>` | Yes | Optional reviewer note |

**`PaymentRecord`**

| Field | Type | Nullable | Description |
|---|---|---|---|
| `paymentId` | `String` | No | Payment identifier |
| `targetApi` | `String` | No | Target service |
| `amountCents` | `long` | No | Amount in cents |
| `status` | `PaymentStatus` | No | Current payment status |
| `approvalRequired` | `boolean` | No | Whether this payment required HITL |
| `reviewedBy` | `Optional<String>` | Yes | Reviewer identity if approved/rejected |
| `settledAt` | `Optional<Instant>` | Yes | Timestamp of settlement or rejection |

### Events on `PaymentTaskEntity`

| Event | Trigger |
|---|---|
| `TaskCreated` | `PaymentTaskEndpoint` submits a new task |
| `AgentStarted` | `PaymentAgentSession` begins processing |
| `PaymentIntentSubmitted` | Agent submits a payment intent; entity evaluates threshold |
| `PaymentHeld` | Intent amount ≥ threshold; workflow started |
| `PaymentApproved` | Workflow receives `APPROVE` from reviewer |
| `PaymentRejected` | Workflow receives `REJECT` from reviewer |
| `PaymentSettled` | Payment recorded as executed (below threshold or post-approval) |
| `BudgetCapReached` | Accumulated spend + pending intent would breach budget |
| `TaskCompleted` | Agent signals it has finished all intended payments within budget |
| `TaskFailed` | Unrecoverable error during agent session |

### Status enums

**`TaskStatus`**: `PENDING` | `ACTIVE` | `AWAITING_APPROVAL` | `BUDGET_EXCEEDED` | `COMPLETED` | `FAILED`

**`PaymentStatus`**: `PENDING_APPROVAL` | `APPROVED` | `REJECTED` | `SETTLED` | `CANCELLED`

**`ApprovalStatus`**: `APPROVE` | `REJECT`

---

## 6. API contract

| Method | Path | Description |
|---|---|---|
| `POST` | `/tasks` | Submit a new payment task |
| `GET` | `/tasks` | List all tasks (newest first) |
| `GET` | `/tasks/{taskId}` | Get a single task with full payment history |
| `GET` | `/tasks/{taskId}/stream` | SSE stream for real-time task updates |
| `POST` | `/tasks/{taskId}/approvals` | Submit an approval decision for a held payment |
| `GET` | `/metadata/readme` | Serve `README.md` |
| `GET` | `/metadata/risk-survey` | Serve `risk-survey.yaml` |
| `GET` | `/metadata/eval-matrix` | Serve `eval-matrix.yaml` |
| `GET` | `/` and `/app/*` | Serve static UI |

---

## 7. Agent behavior

### `PaymentAgentSession`

The agent receives the task description, budget, and approval threshold. It decomposes the task into discrete payment intents and submits them one at a time to `PaymentTaskEntity`.

For each payment intent:
1. The agent calls `submitPaymentIntent` on the entity.
2. If the entity accepts the intent (below threshold, within budget), the payment is settled immediately and the agent proceeds.
3. If the entity holds the intent (above threshold), the agent waits for the approval workflow to resolve before continuing.
4. If the entity signals budget cap reached, the agent stops immediately without submitting further intents.

The agent uses its tool call budget to limit total LLM iterations. If the budget is exhausted before the task description is fully satisfied, the agent emits a `TaskCompleted` event with a partial summary.

### Tool calls available to agent

| Tool | Effect |
|---|---|
| `submitPaymentIntent` | Submit a payment intent to the entity |
| `getTaskStatus` | Read current task state including accumulated spend |
| `listPendingApprovals` | List payments currently awaiting human approval |

---

## 8. Governance controls

### Control 1 — HITL (application)

**Trigger:** `PaymentTaskEntity` receives a `PaymentIntentSubmitted` command where `amountCents >= approvalThresholdCents`.

**Mechanism:** The entity emits `PaymentHeld` and starts a `PaymentApprovalWorkflow`. The workflow waits (up to a configurable timeout) for a human decision via `POST /tasks/{taskId}/approvals`. The agent is blocked from submitting additional intents for this payment until the workflow resolves.

**On approval:** The workflow calls back to the entity with `approvePayment`, the entity emits `PaymentApproved` then `PaymentSettled`, and the agent resumes.

**On rejection:** The workflow calls back with `rejectPayment`, the entity emits `PaymentRejected`, and the agent can choose to continue with a lower-cost alternative or complete the task.

**On timeout:** The workflow emits a rejection after the deadline, treating the payment as rejected.

### Control 2 — Halt (budget-cap)

**Trigger:** `PaymentTaskEntity` evaluates a `PaymentIntentSubmitted` command and finds that `accumulatedSpentCents + amountCents >= budgetCents`.

**Mechanism:** The entity emits `BudgetCapReached` and transitions to `BUDGET_EXCEEDED`. The agent session receives a halt signal on its next tool call and stops without emitting further intents. Any in-flight approval workflow for a pending payment is also terminated.

**Recovery:** The task remains in `BUDGET_EXCEEDED` state permanently. A new task must be submitted if additional work is required.

---

## 9. Non-functional requirements

| Requirement | Target |
|---|---|
| Payment intent to settlement latency (below threshold) | < 500 ms p99 |
| Approval workflow timeout | Configurable; default 24 hours |
| Concurrent tasks per service instance | ≥ 50 |
| Event log retention | Indefinite (event-sourced) |
| SSE heartbeat interval | 15 seconds |
| Agent tool call budget | 20 iterations per task |

---

## 10. Open questions

1. Should rejected payments count toward accumulated spend for budget tracking? (Current decision: no — only settled payments count.)
2. Should the approval timeout trigger a task failure or a soft rejection? (Current decision: soft rejection, agent continues.)
3. What external notification mechanism should alert reviewers when a payment is held? (Out of scope for this sample — deployer responsibility.)

---

## 11. Identity

| Field | Value |
|---|---|
| Folder | `single-agent.ops-automation.agentcore-payments` |
| Maven group | `io.akka.samples` |
| Maven artifact | `single-agent-ops-automation-agentcore-payments` |
| Java package | `io.akka.samples.agentinitiatedpaymentsettlement` |
| Akka version | `3.6.0` |
| HTTP port | `9164` |

---

## 12. Constraints

The generated implementation must apply the following lessons:

- **Lesson 1** — Entity commands return effects; never mutate state directly.
- **Lesson 4** — Workflow steps use explicit timeouts; no open-ended waits.
- **Lesson 6** — All nullable fields use `Optional<T>`; never use raw `null` in records.
- **Lesson 7** — The agent session is a stateless orchestrator; durable state lives in the entity.
- **Lesson 8** — Tool call results are validated before the agent continues.
- **Lesson 9** — Budget cap enforcement is synchronous inside the entity command handler; no async check.
- **Lesson 10** — The approval workflow is idempotent; replaying the same decision is a no-op.
- **Lesson 11** — SSE view is backed by a key-value projection; no polling from the endpoint.
- **Lesson 12** — Events carry full payloads; the state is reconstructed from events alone.
- **Lesson 13** — Approval timeout is handled inside the workflow, not by an external cron.
- **Lesson 23** — The agent never calls an external service directly; all side effects go through entity commands.
- **Lesson 24** — Mermaid state diagrams use CSS class overrides for state label colours.
- **Lesson 25** — The halt control is enforced at the entity layer, not in the agent prompt.
- **Lesson 26** — UI tab switching uses `data-tab` / `data-panel` attribute matching, never NodeList index.

---

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
