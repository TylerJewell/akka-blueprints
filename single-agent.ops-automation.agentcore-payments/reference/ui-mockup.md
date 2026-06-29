# UI Mockup: Agent-Initiated Payment Settlement

The UI is a single-page application served from `GET /` and `GET /app/*`. It has five tabs.

## Tab switching — implementation requirement

Tab switching MUST use attribute-based matching. Each tab button carries a `data-tab` attribute and each panel carries a `data-panel` attribute with the same value. The activation logic must be:

```javascript
document.querySelectorAll('[data-tab]').forEach(btn => {
  btn.addEventListener('click', () => {
    const target = btn.getAttribute('data-tab');
    document.querySelectorAll('[data-tab]').forEach(b =>
      b.toggleAttribute('aria-selected', b === btn));
    document.querySelectorAll('[data-panel]').forEach(p =>
      p.hidden = p.getAttribute('data-panel') !== target);
  });
});
```

Never use NodeList index to match tabs to panels (Lesson 26).

---

## Tab 1 — Overview

```
┌─────────────────────────────────────────────────────────┐
│  Akka Sample: Agent-Initiated Payment Settlement        │
│                                                         │
│  Try it                                                 │
│  ┌──────────────────────────────────────────────────┐   │
│  │  1. Run /akka:build to start the service         │   │
│  │  2. POST /tasks to submit your first task        │   │
│  │  3. Watch payments settle in Tab 5 (App UI)      │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  How it works                                           │
│  The agent receives a task description and spend        │
│  envelope. It decomposes the task into payment intents  │
│  and submits them one at a time. Payments below the     │
│  approval threshold settle immediately; larger payments │
│  are held until a human reviewer approves or rejects    │
│  them. When accumulated spend reaches the budget        │
│  ceiling the agent halts.                               │
│                                                         │
│  Components                                             │
│  ┌──────────────────────────┬────────────────────────┐  │
│  │ Component                │ Role                   │  │
│  ├──────────────────────────┼────────────────────────┤  │
│  │ PaymentTaskEntity        │ Task state + spend     │  │
│  │ PaymentApprovalWorkflow  │ HITL approval loop     │  │
│  │ PaymentAgentSession      │ AI decision-making     │  │
│  │ PaymentLedgerView        │ Read model / SSE       │  │
│  │ PaymentTaskEndpoint      │ HTTP + SSE gateway     │  │
│  └──────────────────────────┴────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

---

## Tab 2 — Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Architecture                                           │
│                                                         │
│  ┌────────────┬────────────┬─────────────┬──────────┐   │
│  │  5 comps   │  2 controls│  10 events  │  SSE     │   │
│  └────────────┴────────────┴─────────────┴──────────┘   │
│                                                         │
│  [Component Graph]    [Sequence Diagram]                │
│  ┌──────────────┐     ┌──────────────┐                  │
│  │ mermaid      │     │ mermaid      │                  │
│  │ flowchart    │     │ sequence     │                  │
│  └──────────────┘     └──────────────┘                  │
│                                                         │
│  [State Machine]      [Entity Model]                    │
│  ┌──────────────┐     ┌──────────────┐                  │
│  │ mermaid      │     │ mermaid      │                  │
│  │ stateDiagram │     │ erDiagram    │                  │
│  └──────────────┘     └──────────────┘                  │
│                                                         │
│  State colours: AWAITING_APPROVAL=amber                 │
│  BUDGET_EXCEEDED=red  COMPLETED=teal  ACTIVE=blue       │
└─────────────────────────────────────────────────────────┘
```

---

## Tab 3 — Risk Survey

Seven sub-tabs render `risk-survey.yaml`:

```
┌─────────────────────────────────────────────────────────┐
│  [Sector] [Decisions] [Data] [Capabilities]             │
│  [Model]  [Oversight] [Deployer fields]                 │
│                                                         │
│  (active: Decisions)                                    │
│  ┌──────────────────────────────────────────────────┐   │
│  │ D-001  Which APIs to call and how much to pay    │   │
│  │        Authority: recommend-and-execute          │   │
│  │        Blast radius: single-task                 │   │
│  │        Reversibility: partially-reversible       │   │
│  │                                                  │   │
│  │ D-002  Whether to continue after rejection       │   │
│  │        Authority: autonomous                     │   │
│  │        Blast radius: single-task                 │   │
│  │        Reversibility: fully-reversible           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  Deployer-specific fields show TO_BE_COMPLETED          │
│  highlighted in amber until filled.                     │
└─────────────────────────────────────────────────────────┘
```

---

## Tab 4 — Eval Matrix

```
┌─────────────────────────────────────────────────────────┐
│  Eval Matrix                                            │
│                                                         │
│  ┌──────┬──────────────────────┬──────────┬────────────┐│
│  │ ID   │ Name                 │ Kind     │ Tier       ││
│  ├──────┼──────────────────────┼──────────┼────────────┤│
│  │ H-001│ Hold above-threshold │ hitl     │ out-of-box ││
│  │      │ payments for...      │          │            ││
│  │ H-002│ Halt agent when      │ halt     │ out-of-box ││
│  │      │ budget exhausted     │          │            ││
│  └──────┴──────────────────────┴──────────┴────────────┘│
│                                                         │
│  Click any row to expand rationale and implementation.  │
│                                                         │
│  [H-001 expanded]                                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Rationale: Payments at or above the task's ...   │   │
│  │ Implementation: Inside PaymentTaskEntity's ...   │   │
│  │ Regulation anchors: none                         │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Tab 5 — App UI

```
┌─────────────────────────────────────────────────────────┐
│  Submit Task              │  Task detail: task-001      │
│  ┌────────────────────┐   │  ┌───────────────────────┐  │
│  │ Task ID            │   │  │ Status: AWAITING_APPR │  │
│  │ [task-001        ] │   │  │ Budget: $50.00        │  │
│  │ Description        │   │  │ Spent:  $8.00  ██░░░░ │  │
│  │ [Fetch firmogr...] │   │  │ Remaining: $42.00     │  │
│  │ Budget (cents)     │   │  └───────────────────────┘  │
│  │ [5000            ] │   │                             │
│  │ Threshold (cents)  │   │  Payments                   │
│  │ [1000            ] │   │  ┌───────────────────────┐  │
│  │ [Submit Task]      │   │  │ pay-001 SETTLED  $8.00 │  │
│  └────────────────────┘   │  │ enrichment-api/firmog  │  │
│                           │  ├───────────────────────┤  │
│  Recent tasks             │  │ pay-002 PENDING  $15.00│  │
│  ┌────────────────────┐   │  │ enrichment-api/deep    │  │
│  │ task-001 AWAITING  │   │  │ [Approve] [Reject]    │  │
│  │ task-002 COMPLETED │   │  └───────────────────────┘  │
│  │ task-003 ACTIVE    │   │                             │
│  └────────────────────┘   │  Reviewer: [ops@ex.com  ]  │
│                           │  Note:     [optional...  ]  │
└─────────────────────────────────────────────────────────┘
```

### Colour scheme

| Status / amount | Colour |
|---|---|
| `SETTLED` | green |
| `PENDING_APPROVAL` | amber |
| `REJECTED` | red |
| `CANCELLED` | grey |
| `BUDGET_EXCEEDED` task status | red badge |
| `COMPLETED` task status | green badge |
| `ACTIVE` / `AWAITING_APPROVAL` task status | blue / amber badge |

### Approval panel

The approval panel (Approve / Reject buttons) is only shown for payments in `PENDING_APPROVAL` status. Clicking Approve or Reject POSTs to `/tasks/{taskId}/approvals` with the reviewer identity from the reviewer field.

### Live updates

Tab 5 opens an EventSource to `GET /tasks/{taskId}/stream` when a task row is selected. All payment status changes and task status transitions update the panel in real time without a page reload.
