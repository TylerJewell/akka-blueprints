# UI Mockup — Financial Model Builder

Five-tab single-page application. Tab switching uses `data-tab` / `data-panel` attributes (no JavaScript class toggling).

```html
<title>Akka Sample: Financial Model Builder</title>

<nav class="sample-tabs">
  <button data-tab="overview"      class="tab-btn active">Overview</button>
  <button data-tab="architecture"  class="tab-btn">Architecture</button>
  <button data-tab="risk-survey"   class="tab-btn">Risk Survey</button>
  <button data-tab="eval-matrix"   class="tab-btn">Eval Matrix</button>
  <button data-tab="app-ui"        class="tab-btn">App UI</button>
</nav>
```

---

## Tab 1 — Overview

```html
<section data-panel="overview" class="tab-panel active">
  <p class="eyebrow">Overview</p>
  <h1>Financial Model <span class="accent">Builder</span></h1>

  <!-- Try it card -->
  <div class="card">
    <h2>Try it</h2>
    <pre><code>/akka:build</code></pre>
    <p>Then open <a href="/app">http://localhost:9286/app</a> and submit a ticker.</p>
  </div>

  <!-- How it works card -->
  <div class="card">
    <h2>How it works</h2>
    <ol>
      <li>Submit <code>{ ticker, period }</code> — <code>FinancialModelEntity</code> is created and the pipeline starts.</li>
      <li><code>FinancialModelPipelineWorkflow</code> calls <code>FinancialModelAgent</code> three times: EXTRACT → BUILD → VALIDATE.</li>
      <li><code>ModelPhaseGuardrail</code> blocks any tool call that belongs to the wrong phase.</li>
      <li>After VALIDATED, the workflow pauses. An analyst approves or rejects via the App UI.</li>
      <li>On approval, <code>FilingFidelityScorer</code> runs and emits a score (1–5).</li>
    </ol>
  </div>

  <!-- Components card -->
  <div class="card">
    <h2>Components</h2>
    <table>
      <tr><td>FinancialModelAgent</td><td>AutonomousAgent</td></tr>
      <tr><td>FinancialModelPipelineWorkflow</td><td>Workflow</td></tr>
      <tr><td>FinancialModelEntity</td><td>EventSourcedEntity</td></tr>
      <tr><td>ExtractTools / BuildTools / ValidateTools</td><td>Tool classes</td></tr>
      <tr><td>ModelPhaseGuardrail</td><td>Guardrail (before-tool-call)</td></tr>
      <tr><td>FilingFidelityScorer</td><td>Scorer (on-decision-eval)</td></tr>
      <tr><td>FinancialModelView</td><td>View</td></tr>
      <tr><td>ModelEndpoint / AppEndpoint</td><td>HTTP endpoints</td></tr>
    </table>
  </div>

  <!-- API card -->
  <div class="card">
    <h2>API contract</h2>
    <p>See <a href="/api/metadata/readme">API contract reference</a> or <code>reference/api-contract.md</code>.</p>
  </div>
</section>
```

---

## Tab 2 — Architecture

```html
<section data-panel="architecture" class="tab-panel">
  <p class="eyebrow">Architecture</p>
  <h1>What gets <span class="accent">wired together</span></h1>

  <!-- Stat tiles -->
  <div class="stat-row">
    <div class="stat-tile"><span class="stat-num">3</span><span class="stat-label">Task phases</span></div>
    <div class="stat-tile"><span class="stat-num">2</span><span class="stat-label">Governance controls</span></div>
    <div class="stat-tile"><span class="stat-num">13</span><span class="stat-label">Entity states</span></div>
    <div class="stat-tile"><span class="stat-num">1</span><span class="stat-label">Agent</span></div>
  </div>

  <!-- Four mermaid cards (content from PLAN.md) -->
  <div class="card"><h2>Component graph</h2><!-- mermaid: component flowchart --></div>
  <div class="card"><h2>Interaction sequence</h2><!-- mermaid: sequence diagram --></div>
  <div class="card"><h2>State machine</h2><!-- mermaid: stateDiagram-v2 --></div>
  <div class="card"><h2>Entity model</h2><!-- mermaid: erDiagram --></div>

  <!-- Component table -->
  <div class="card">
    <h2>Component table</h2>
    <table>
      <thead><tr><th>Java file</th><th>Kind</th><th>Responsibility</th></tr></thead>
      <tbody>
        <tr><td>FinancialModelEntity</td><td>EventSourcedEntity</td><td>State owner; commands → events</td></tr>
        <tr><td>FinancialModelPipelineWorkflow</td><td>Workflow</td><td>Pipeline orchestration</td></tr>
        <tr><td>FinancialModelAgent</td><td>AutonomousAgent</td><td>One task per invocation</td></tr>
        <tr><td>ExtractTools</td><td>Tool class</td><td>parseFiling, fetchLineItem</td></tr>
        <tr><td>BuildTools</td><td>Tool class</td><td>computeRatio, projectFigure</td></tr>
        <tr><td>ValidateTools</td><td>Tool class</td><td>crossCheckAssumption, detectAnomalies</td></tr>
        <tr><td>ModelPhaseGuardrail</td><td>Guardrail</td><td>before-tool-call phase gate</td></tr>
        <tr><td>FilingFidelityScorer</td><td>Scorer</td><td>on-decision-eval; 4 checks; score 1–5</td></tr>
        <tr><td>FinancialModelView</td><td>View</td><td>SSE-ready read model</td></tr>
        <tr><td>ModelEndpoint</td><td>HttpEndpoint</td><td>/api/models/*</td></tr>
        <tr><td>AppEndpoint</td><td>HttpEndpoint</td><td>/app/* + / redirect</td></tr>
      </tbody>
    </table>
  </div>
</section>
```

---

## Tab 3 — Risk Survey

```html
<section data-panel="risk-survey" class="tab-panel">
  <p class="eyebrow">Risk Survey</p>
  <h1>Risk profile for <span class="accent">this deployment</span></h1>

  <!-- 7 sub-tabs rendering risk-survey.yaml sections -->
  <nav class="subtabs">
    <button data-tab="rs-purpose"    class="subtab-btn active">Purpose</button>
    <button data-tab="rs-data"       class="subtab-btn">Data</button>
    <button data-tab="rs-decisions"  class="subtab-btn">Decisions</button>
    <button data-tab="rs-failure"    class="subtab-btn">Failure</button>
    <button data-tab="rs-oversight"  class="subtab-btn">Oversight</button>
    <button data-tab="rs-operations" class="subtab-btn">Operations</button>
    <button data-tab="rs-compliance" class="subtab-btn">Compliance</button>
  </nav>

  <div data-panel="rs-purpose"    class="subtab-panel active"><!-- render purpose section --></div>
  <div data-panel="rs-data"       class="subtab-panel"><!-- render data section --></div>
  <div data-panel="rs-decisions"  class="subtab-panel"><!-- render decisions section --></div>
  <div data-panel="rs-failure"    class="subtab-panel"><!-- render failure section --></div>
  <div data-panel="rs-oversight"  class="subtab-panel"><!-- render oversight section --></div>
  <div data-panel="rs-operations" class="subtab-panel"><!-- render operations section --></div>
  <div data-panel="rs-compliance" class="subtab-panel"><!-- render compliance section --></div>
</section>
```

---

## Tab 4 — Eval Matrix

```html
<section data-panel="eval-matrix" class="tab-panel">
  <p class="eyebrow">Eval Matrix</p>
  <h1>Controls the <span class="accent">runtime enforces</span></h1>

  <table class="eval-matrix-table">
    <thead>
      <tr>
        <th>ID</th>
        <th>Name</th>
        <th>Mechanism</th>
        <th>Enforcement</th>
        <th>Implementation</th>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>H1</td>
        <td>Analyst approval gate</td>
        <td><span class="badge badge--hitl">HITL</span></td>
        <td><span class="badge badge--blocking">blocking</span></td>
        <td>application</td>
      </tr>
      <tr>
        <td>E1</td>
        <td>Filing fidelity eval</td>
        <td><span class="badge badge--eval">Eval · event</span></td>
        <td><span class="badge badge--nonblocking">non-blocking</span></td>
        <td>on-decision-eval</td>
      </tr>
    </tbody>
  </table>
</section>
```

---

## Tab 5 — App UI

```html
<section data-panel="app-ui" class="tab-panel">
  <p class="eyebrow">App UI</p>
  <h1>Submit a ticker. <span class="accent">Review the model.</span></h1>
  <p class="subtitle">One agent, three task phases, one analyst approval gate.</p>

  <!-- Two-column layout -->
  <div class="app-ui-layout">

    <!-- Left column: live model list -->
    <aside class="model-list">
      <h2>Models</h2>
      <!-- Submit form -->
      <form class="submit-form">
        <input type="text" name="ticker" placeholder="Ticker (e.g. AAPL)" />
        <input type="text" name="period" placeholder="Period (e.g. 2025-Q4)" />
        <button type="submit">Submit</button>
      </form>
      <!-- Model cards; populated via SSE on /api/models/sse -->
      <div class="model-cards">
        <!-- Each card shows: -->
        <!-- - status pill (colour-coded) -->
        <!-- - eval score chip (1-5, shown when EVALUATED) -->
        <!-- - ticker + period -->
        <!-- - age (createdAt relative) -->
        <!-- - red dot if GuardrailRejected event exists -->
      </div>
    </aside>

    <!-- Right column: selected model detail -->
    <main class="model-detail">

      <!-- EXTRACT panel -->
      <section class="phase-panel" data-phase="extract">
        <h3>Extracted line items</h3>
        <table>
          <thead><tr><th>Name</th><th>Period</th><th>Value</th><th>Unit</th><th>Source ref</th></tr></thead>
          <tbody><!-- populated from filingData.items --></tbody>
        </table>
      </section>

      <!-- BUILD panel -->
      <section class="phase-panel" data-phase="build">
        <h3>Model rows</h3>
        <table>
          <thead><tr><th>Label</th><th>Period</th><th>Value</th><th>Derivation</th></tr></thead>
          <tbody><!-- populated from model.rows --></tbody>
        </table>
        <h3>Assumptions</h3>
        <ul class="assumption-list"><!-- populated from model.assumptions --></ul>
      </section>

      <!-- VALIDATE panel -->
      <section class="phase-panel" data-phase="validate">
        <h3>Validation flags</h3>
        <ul class="flag-list">
          <!-- Each flag shows severity badge: CRITICAL=red, WARNING=yellow, INFO=blue -->
        </ul>
        <p class="confidence-line">Confidence: <strong><!-- confidence --></strong></p>
      </section>

      <!-- REVIEW panel — shown only when status = PENDING_REVIEW -->
      <section class="phase-panel phase-panel--review" data-phase="review">
        <h3>Analyst review</h3>
        <textarea name="analystNote" placeholder="Optional note for the record..."></textarea>
        <div class="review-actions">
          <button class="btn btn--approve">Approve</button>
          <button class="btn btn--reject">Reject</button>
        </div>
      </section>

      <!-- EVAL panel -->
      <section class="phase-panel" data-phase="eval">
        <h3>Filing fidelity score</h3>
        <span class="score-chip"><!-- score 1-5 --></span>
        <p class="eval-rationale"><!-- rationale --></p>
      </section>

    </main>
  </div>
</section>
```

### Status pill colour reference

| Status | Colour |
|---|---|
| CREATED | grey |
| EXTRACTING / BUILDING / VALIDATING | blue (animated) |
| EXTRACTED / BUILT / VALIDATED | blue |
| PENDING_REVIEW | orange |
| APPROVED | green |
| REJECTED | red |
| EVALUATED | teal |
| FAILED | red |

### CSS notes

- Apply Lesson 24 token overrides for the dark-themed mermaid diagrams (see PLAN.md theme block).
- The App UI tab consumes SSE from `GET /api/models/sse` using `EventSource` — no polling (Lesson 25).
- Tab and sub-tab switching is driven by `data-tab` / `data-panel` attribute selectors in CSS; JavaScript only adds/removes the `active` attribute value (Lesson 26).
- Do not apply local `.btn--primary` overrides that produce a dark-fill button with dark text — let the global stylesheet win (Lesson 24 CSS correctness).
