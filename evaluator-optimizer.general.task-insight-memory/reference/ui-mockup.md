# UI mockup — task-insight-memory

The generated `static-resources/index.html` is a single self-contained file with inline CSS and JS. No build step, no `ui/` folder. Five tabs; tab switching is attribute-based (`data-tab` / `data-panel`) per Lesson 26.

## Global structure

```html
<html lang="en">
<head>
  <title>Akka Sample: Task-Centric Memory</title>
  <!-- inline CSS; CDN imports for mermaid, js-yaml, marked -->
</head>
<body>
  <header class="app-header">…logo + nav tabs…</header>
  <main>
    <nav class="tab-bar" role="tablist">
      <button data-tab="overview"      role="tab">Overview</button>
      <button data-tab="architecture"  role="tab">Architecture</button>
      <button data-tab="risk-survey"   role="tab">Risk Survey</button>
      <button data-tab="eval-matrix"   role="tab">Eval Matrix</button>
      <button data-tab="app-ui"        role="tab">App UI</button>
    </nav>
    <section class="tab-panel" data-panel="overview">…</section>
    <section class="tab-panel" data-panel="architecture">…</section>
    <section class="tab-panel" data-panel="risk-survey">…</section>
    <section class="tab-panel" data-panel="eval-matrix">…</section>
    <section class="tab-panel" data-panel="app-ui">…</section>
  </main>
</body>
</html>
```

Tab switching logic:

```js
document.querySelectorAll('[data-tab]').forEach(btn => {
  btn.addEventListener('click', () => {
    const target = btn.dataset.tab;
    document.querySelectorAll('[data-panel]').forEach(p => {
      p.hidden = p.dataset.panel !== target;
    });
    document.querySelectorAll('[data-tab]').forEach(b => {
      b.setAttribute('aria-selected', b.dataset.tab === target ? 'true' : 'false');
    });
  });
});
```

## Tab 1 — Overview

```
eyebrow:  Overview
headline: Task-Centric Memory
(no subtitle)

Four cards in a 2 × 2 grid:

┌─────────────────────────────────┬─────────────────────────────────┐
│  Try it                         │  How it works                   │
│  Submit a task → watch it       │  ExecutorAgent retrieves        │
│  execute → see the insight      │  prior insights, runs the task, │
│  stored or rejected in real     │  EvaluatorAgent scores the      │
│  time.                          │  result, PII is scrubbed, and   │
│                                 │  verified results are persisted  │
│  [Submit a task ↗]              │  into the memory store.         │
├─────────────────────────────────┼─────────────────────────────────┤
│  Components                     │  API contract                   │
│  ExecutorAgent                  │  POST /api/tasks                │
│  EvaluatorAgent                 │  GET  /api/tasks                │
│  MemoryWorkflow                 │  GET  /api/tasks/{id}           │
│  MemoryEntity / TaskEntity      │  GET  /api/tasks/sse            │
│  MemoryView / TaskView          │  GET  /api/memory               │
│  TaskConsumer                   │  POST /api/memory/correction    │
│  TaskSimulator / DriftWatcher   │  POST /api/memory/demonstration │
└─────────────────────────────────┴─────────────────────────────────┘
```

## Tab 2 — Architecture

Four mermaid diagrams rendered in sequence with narrative text beside each (see `reference/architecture.md`). Requires the mermaid CSS overrides from Lesson 24:

```html
<style>
  /* Lesson 24: state-diagram label colour override */
  .statediagram-state .label { color: #cccccc !important; }
  /* edge-label foreignObject overflow */
  .edgeLabel foreignObject { overflow: visible !important; }
</style>
<script>
  mermaid.initialize({
    theme: 'base',
    themeVariables: {
      primaryColor:       '#1a1a2e',
      primaryTextColor:   '#e0e0e0',
      primaryBorderColor: '#7EC8E3',
      lineColor:          '#7EC8E3',
      secondaryColor:     '#16213e',
      tertiaryColor:      '#0f3460',
      transitionLabelColor: '#cccccc'
    }
  });
</script>
```

A click-to-expand component table below the diagrams lists all 13 components with their Akka primitive and role.

## Tab 3 — Risk Survey

Seven sub-tabs rendered from `risk-survey.yaml`. Sub-tab IDs: `purpose`, `data`, `decisions`, `failure`, `oversight`, `operations`, `compliance`. Deployer-specific fields (`TO_BE_COMPLETED_BY_DEPLOYER`) render with `opacity: 0.45` and a light italic placeholder. The sub-tab switcher is also attribute-based (`data-subtab` / `data-subpanel`).

## Tab 4 — Eval Matrix

Five-column table: ID / Control / Mechanism / Implementation / Source.

| ID | Control | Mechanism | Implementation | Source |
|----|---------|-----------|----------------|--------|
| E1 | Record every task outcome as an eval event | Eval · event | on-decision-eval | — |
| S1 | Scrub PII from insight candidates | Sanitizer | before-persist | — |
| P1 | Watch accumulated memory for drift | Eval · periodic | drift-fairness-watch | — |

ID badges are coloured by mechanism: eval-event = blue (`#3b82f6`), sanitizer = orange (`#f97316`), eval-periodic = blue (`#3b82f6`). Click any row to expand the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

Two panels side by side at ≥ 900 px, stacked on narrow screens.

**Left panel — Task submission and live task list**

```
┌─────────────────────────────────────────────────────┐
│  Submit a task                                       │
│  Task type:  [data-extraction          ▼]           │
│  Description: [                            ]        │
│  Acceptance criteria: [                    ]        │
│  [Submit task]                                       │
├─────────────────────────────────────────────────────┤
│  Tasks                            Filter: [All ▼]   │
│                                                     │
│  ● t-9b3c1…  data-extraction   VERIFIED  09:14      │
│    ▶ Expand                                         │
│                                                     │
│  ● t-2a8e4…  summarization     EXECUTING 09:13      │
│    ▶ Expand                                         │
│                                                     │
│  ● t-7f1c9…  classification    FAILED    09:12      │
│    ▶ Expand                                         │
└─────────────────────────────────────────────────────┘
```

Expanding a task row reveals:
- Retrieved insights used (list of `insightId`, `text`, `confidence`, `provenance` pill).
- Task result: `answer`, `confidence` gauge, `keyFindings` list.
- Eval verdict: `outcome` pill (`VERIFIED` green / `REJECTED` red), `qualityScore`, `notes.bullets`, `notes.overallRationale`.
- Insight persisted: insightId if written; "Not persisted" if rejected or below confidence threshold.

Status pills: `PENDING` grey, `EXECUTING` blue (animated), `EVALUATED` yellow, `VERIFIED` green, `FAILED` red.

**Right panel — Memory store**

```
┌─────────────────────────────────────────────────────┐
│  Memory store                Filter: [All types ▼]  │
│  12 insights · 3 task types                          │
├─────────────────────────────────────────────────────┤
│  data-extraction (7)                                │
│  ├─ [VE] Paragraphs mix %, absolute figures…  0.92  │
│  ├─ [VE] Headcount appears as 'from X to Y'…  0.88  │
│  ├─ [CO] Always check implied units…          0.95  │
│  └─ …                                               │
│                                                     │
│  summarization (4)                                  │
│  ├─ [VE] Main finding often in paragraph 4…   0.81  │
│  └─ …                                               │
│                                                     │
│  ⚠ Drift alert  09:20  TYPE_CONCENTRATION 0.83      │
└─────────────────────────────────────────────────────┘
```

Provenance pills: `VE` = VERIFIED_EXPERIENCE (blue), `CO` = CORRECTION (orange), `DM` = DEMONSTRATION (purple). Drift alert rows are amber with the `DriftReason` metrics inline. Superseded insights are hidden by default; a "show superseded" toggle reveals them with strikethrough styling.

Live updates for both panels are driven by the SSE stream on `GET /api/tasks/sse`. The memory panel polls `GET /api/memory` on each task `VERIFIED` event to refresh the insight list. Content fits the 1080 px column with no horizontal scroll (Lesson 12).
