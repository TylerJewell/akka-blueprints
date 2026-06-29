# UI mockup — brand-aligner

The generated system ships a single self-contained `static-resources/index.html`. Five tabs; no external build step; inline CSS and JS; runtime CDN for markdown and YAML rendering.

Browser title: `<title>Akka Sample: Brand Aligner</title>`.

---

## Tab structure

Tab switching is attribute-based. Each tab button carries `data-tab="<name>"` and each panel carries `data-panel="<name>"`. The activation script finds the target panel with `document.querySelector('[data-panel="' + tab + '"]')` — never by NodeList index (Lesson 26).

The DOM contains exactly five `<section class="tab-panel">` elements:

```html
<nav class="tab-bar">
  <button data-tab="overview"      class="tab-btn active">Overview</button>
  <button data-tab="architecture"  class="tab-btn">Architecture</button>
  <button data-tab="risk-survey"   class="tab-btn">Risk Survey</button>
  <button data-tab="eval-matrix"   class="tab-btn">Eval Matrix</button>
  <button data-tab="app-ui"        class="tab-btn">App UI</button>
</nav>

<section class="tab-panel" data-panel="overview">   … </section>
<section class="tab-panel" data-panel="architecture"> … </section>
<section class="tab-panel" data-panel="risk-survey">  … </section>
<section class="tab-panel" data-panel="eval-matrix">  … </section>
<section class="tab-panel" data-panel="app-ui">       … </section>
```

Any panel that is removed from the design must be deleted from the HTML, not hidden with `display:none`.

---

## Tab 1 — Overview

- Eyebrow: `<p class="eyebrow">Overview</p>`
- Headline: `<h1>Brand Aligner</h1>`
- No subtitle.
- Four cards in a 2×2 grid:
  - **Try it** — "Submit a campaign brief in the App UI tab. The Copywriter generates a variant; the Brand Reviewer scores it; the two iterate until the copy is approved or the retry ceiling is hit."
  - **How it works** — "An AlignmentWorkflow alternates CopywriterAgent (generate / revise) and BrandReviewerAgent (score) with a deterministic compliance check between them. Each cycle is recorded as a BrandEvalRecorded event. The loop halts gracefully at maxAttempts."
  - **Components** — collapsible table listing all 11 components (class, primitive, one-line role). Default collapsed.
  - **API contract** — collapsible table of all 9 endpoints (method, path, one-line description). Default collapsed.

---

## Tab 2 — Architecture

Four mermaid diagrams rendered in order, each preceded by its narrative heading from `reference/architecture.md`:

1. **Component graph** — the `flowchart TB` diagram from PLAN.md.
2. **Interaction sequence** — the `sequenceDiagram` from PLAN.md.
3. **State machine** — the `stateDiagram-v2` from PLAN.md.
4. **Entity model** — the `erDiagram` from PLAN.md.

Mermaid theme variables and CSS overrides (Lesson 24) are inlined in a `<style>` block:

```css
/* Lesson 24 — state-diagram label colour and edge-label overflow */
:root {
  --mermaid-font-family: ui-monospace, monospace;
}
.stateDiagram-v2 .stateLabel { fill: #cccccc !important; }
.edgeLabel foreignObject { overflow: visible !important; }
.transitionLabel { color: #cccccc; }
```

A click-to-expand component table (same data as the Overview card) follows the diagrams.

---

## Tab 3 — Risk Survey

Seven sub-tabs matching the `governance.html` style:

| Sub-tab | Fields shown |
|---|---|
| Purpose | `primary_function`, `sector`, `decisions_surface`, `user_type` |
| Data | `inputs`, `data_classes.*`, `data_retention_days`, `data_residency` |
| Decisions | `authority_level`, `decision_subject`, `decisions_per_day`, `reversibility` |
| Failure | `blast_radius`, `failure_modes[]`, `acceptable_downtime_minutes` |
| Oversight | `human_in_loop`, `human_on_loop`, `reviewer_role`, `audit_frequency` |
| Operations | `model_family`, `model_version_pinned`, `external_tool_calls[]`, `rate_limit_per_user` |
| Compliance | `capabilities.*`, `jurisdictions_in_scope`, `data-protection-officer_assigned` |

Fields pre-filled from `risk-survey.yaml` render at full opacity. Fields with value `TO_BE_COMPLETED_BY_DEPLOYER` render at opacity 0.45 with the label "(deployer completes)".

Sub-tab switching uses the same attribute-based pattern (`data-subtab` / `data-subpanel`).

---

## Tab 4 — Eval Matrix

A 5-column table with click-to-expand rows. Columns:

| Column | Content |
|---|---|
| ID | coloured badge — blue for `eval-event`, red for `guardrail` / `halt` |
| Control | `name` field (active-voice sentence) |
| Mechanism | `mechanism.kind` + `mechanism.hook` or `mechanism.flavor` |
| Implementation | `implementation` paragraph (collapsed to one line; expand on click) |
| Source | `regulation_anchors[]` (shown as "—" when empty) |

Controls from `eval-matrix.yaml`: `CC1` (red), `E1` (blue), `HT1` (red).

---

## Tab 5 — App UI

### Form

```
Topic:           [ free-text input, placeholder "Describe the campaign or product…" ]
Target Audience: [ free-text input, placeholder "e.g. backend engineers"            ]
Word ceiling:    [ number input, default 150, min 10, max 1000                       ]
                 [ Submit button ]
```

Submits `POST /api/materials`. On success, prepends a new card to the list. On error, shows an inline red message.

### Live list

Connected to `GET /api/materials/sse`. Each event with `event: material-update` updates the corresponding card by `materialId`. A fresh page-load fetches `GET /api/materials` to populate the initial list.

Each card shows:
- Material topic (bold) + target audience (muted)
- Status pill: `DRAFTING` (grey), `REVIEWING` (yellow), `APPROVED` (green), `REJECTED_FINAL` (red)
- Attempt count: "Attempt N / maxAttempts"
- Click to expand → per-attempt timeline

### Per-attempt timeline (expanded)

For each `VariantAttempt`:

```
Attempt N
  Variant:     [ full text, scrollable ]
  Word count:  N words
  Compliance:  [ OK pill (green) | OVER_WORD_CEILING pill (red) + detail ]
  Review:      [ APPROVE pill (green) | REVISE pill (yellow) ]
    Score:     N / 5
    Notes:     [ bullet list ]
    Rationale: [ one-sentence overallRationale ]
  Eval events: [ BrandEvalRecorded entries for this attempt ]
```

The terminal block (shown after the last attempt):
- `APPROVED` → "Approved on attempt N" + full approved text.
- `REJECTED_FINAL` → "Rejected after N attempts" + best-of text + rejection reason.
