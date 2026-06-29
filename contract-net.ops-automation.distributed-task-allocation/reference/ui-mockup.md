# UI Mockup — Contract-Net Task Auctioneer

The UI is a single-page application served at `http://localhost:9202`. It has five tabs. Tab switching uses data-attribute selectors on a `<nav>` element; no JavaScript class toggling is used (Lesson 26).

```html
<nav data-tabset="main">
  <button data-tab="overview"     data-active="true">Overview</button>
  <button data-tab="architecture" data-active="false">Architecture</button>
  <button data-tab="risk-survey"  data-active="false">Risk Survey</button>
  <button data-tab="eval-matrix"  data-active="false">Eval Matrix</button>
  <button data-tab="app-ui"       data-active="false">App UI</button>
</nav>

<section data-tabpanel="overview"     data-visible="true">…</section>
<section data-tabpanel="architecture" data-visible="false">…</section>
<section data-tabpanel="risk-survey"  data-visible="false">…</section>
<section data-tabpanel="eval-matrix"  data-visible="false">…</section>
<section data-tabpanel="app-ui"       data-visible="false">…</section>
```

CSS shows the active panel via `[data-visible="true"]` and hides others via `[data-visible="false"] { display: none; }`. The JavaScript tab switcher sets `data-active` on buttons and `data-visible` on panels — it never touches `classList`.

---

## Tab 1 — Overview

**Purpose:** Explain what the sample demonstrates and how to navigate it.

**Layout:**
```
┌─────────────────────────────────────────────────────────┐
│  Contract-Net Task Auctioneer                           │
│  Pattern: Contract-Net  |  Domain: Ops Automation       │
├─────────────────────────────────────────────────────────┤
│  WHAT THIS SAMPLE SHOWS                                 │
│  ┌─────────────────────┐  ┌─────────────────────────┐  │
│  │ Task Lifecycle      │  │ Governance Controls     │  │
│  │ Announce → Bid      │  │ G-001: Bid guardrail    │  │
│  │ → Award → Execute   │  │ E-001: Perf scoring     │  │
│  │ → Score             │  │                         │  │
│  └─────────────────────┘  └─────────────────────────┘  │
│                                                         │
│  HOW TO USE                                             │
│  1. Register contractors on the App UI tab              │
│  2. Announce a task; watch the auction board update     │
│  3. Submit bids; watch the bid count increment via SSE  │
│  4. Wait for the bidding window to close; observe the   │
│     evaluator and guardrail steps in the event log      │
│  5. Report completion; observe the performance score    │
└─────────────────────────────────────────────────────────┘
```

---

## Tab 2 — Architecture

**Purpose:** Render the four PLAN.md Mermaid diagrams with labels.

**Layout:**
```
┌─────────────────────────────────────────────────────────┐
│  Architecture                                           │
├──────────────────┬──────────────────────────────────────┤
│  Component Graph │  Sequence Diagram                    │
│  [Mermaid §1]    │  [Mermaid §2]                        │
├──────────────────┼──────────────────────────────────────┤
│  State Machine   │  Entity-Relationship                 │
│  [Mermaid §3]    │  [Mermaid §4]                        │
└──────────────────┴──────────────────────────────────────┘
```

Each diagram renders in its own `<figure>` with a `<figcaption>` matching the PLAN.md section title.

---

## Tab 3 — Risk Survey

**Purpose:** Render `risk-survey.yaml` as a structured form. Pre-filled fields display as read-only with a badge **Pre-filled**. `TO_BE_COMPLETED_BY_DEPLOYER` fields render as editable text inputs with a badge **Required**.

**Layout:**
```
┌─────────────────────────────────────────────────────────┐
│  Risk Survey                        [Export YAML]       │
├─────────────────────────────────────────────────────────┤
│  Section 1: Sector & Context                            │
│  sector                 ops-automation    [Pre-filled]  │
│  deployment_environment _______________   [Required]    │
│  geographic_regions     _______________   [Required]    │
│                                                         │
│  Section 2: Decision Types                              │
│  D-01  Award bid        reversible: yes   [Pre-filled]  │
│  D-02  Throttle         reversible: yes   [Pre-filled]  │
│  D-03  Renegotiate      …                [Required]     │
│                                                         │
│  Section 3–7 …                                         │
└─────────────────────────────────────────────────────────┘
```

The **Export YAML** button serializes the form values back to a YAML string for download.

---

## Tab 4 — Eval Matrix

**Purpose:** Render `eval-matrix.yaml` as two control cards with expandable sections.

**Layout:**
```
┌─────────────────────────────────────────────────────────┐
│  Eval Matrix                                            │
├─────────────────────────────────────────────────────────┤
│  ┌───────────────────────────────────────────────────┐  │
│  │ G-001  Validate winning bid before award    [▼]   │  │
│  │ Kind: guardrail  Hook: before-bid-acceptance      │  │
│  │ Tier: runs-out-of-the-box  Audit: BidValidated    │  │
│  │ ─────────────────────────────────────────────     │  │
│  │ [Rationale] [Enforcement] [Implementation]        │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ E-001  Score contractor after task end      [▼]   │  │
│  │ Kind: eval-event  Hook: on-completion-eval        │  │
│  │ Tier: runs-out-of-the-box  Audit: PerformanceScored│ │
│  │ ─────────────────────────────────────────────     │  │
│  │ [Rationale] [Enforcement] [Implementation]        │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

Each `[▼]` toggle expands to show the full text fields. The expanded sub-tabs (Rationale / Enforcement / Implementation) switch via `data-subtab` attributes on the inner `<nav>` within each card.

---

## Tab 5 — App UI

**Purpose:** Live interaction with the running service. Three panels, left-to-right.

**Layout:**
```
┌──────────────────┬──────────────────┬──────────────────┐
│  AUCTION BOARD   │  TASK DETAIL     │  LEADERBOARD     │
│  (SSE stream)    │  (selected task) │  (SSE stream)    │
│                  │                  │                  │
│  [+ New Task]    │  Task: <title>   │  Rank  Name  Sc  │
│                  │  Status: BIDDING │  ─────────────── │
│  ○ Task A  3 bids│  Bids: 3         │  1  Omega   91   │
│  ○ Task B  1 bid │                  │  2  Apex    78   │
│  ● Task C  DONE  │  [Submit Bid]    │  3  Vertex  55 ⚠ │
│                  │  [Mark Complete] │                  │
│                  │  [Mark Failed]   │  ⚠ = throttled   │
│                  │                  │                  │
│                  │  EVENT LOG       │                  │
│                  │  ── TaskAnnounced│                  │
│                  │  ── BidSubmitted │                  │
│                  │  ── BidValidated │                  │
│                  │  ── TaskAwarded  │                  │
└──────────────────┴──────────────────┴──────────────────┘
```

- The Auction Board streams from `GET /auctions/active`.
- Clicking a task row loads its detail from `GET /tasks/{taskId}` and streams its event log from a filtered view subscription.
- The Leaderboard streams from `GET /contractors/leaderboard`. Throttled contractors display a `⚠` badge.
- **New Task** opens a modal form that POSTs to `/tasks`.
- **Submit Bid** opens a modal pre-populated with the current user's contractor ID (set in a session input at page top).
- **Register Contractor** button in the Leaderboard panel opens a modal that POSTs to `/contractors/register`.
