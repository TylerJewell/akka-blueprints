# UI mockup — job-posting-eeo

The generated UI follows the formal exemplar's 5-tab structure, the Akka visual palette (dark / yellow `#F5C518`, Instrument Sans font, dot-grid background), and the language rules from `BLUEPRINT-AUTHORING-GUIDE.md` Section 13.

Browser title: `<title>Akka Sample: Job Posting Team</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. Canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Job Posting <span class="accent">Team</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then three numbered steps (submit company + role, watch the posting progress to CLEARED, expand it for culture / role / draft / sanitizer terms).
- Card **How it works**: one paragraph naming the components and the parallel fan-out.
- Card **Components**: table with rows for each component in `SPEC.md §4`. Kind column coloured per the component palette (agent = blue, workflow = purple, ese = yellow, view = green, consumer = orange, timed = muted, endpoint = white).
- Card **API contract**: table with Path / What it does columns from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`.
- Stat tiles: count of each component kind.
- Four mermaid cards (component graph, sequence, state, ER) with Akka theme variables **and** the Lesson 24 CSS overrides (state-diagram label colour `#ffffff`, edge-label `foreignObject` `overflow:visible`, `transitionLabelColor #cccccc`).
- Compressed comp-row table: one row per component, click expands to show the description plus a short syntax-highlighted Java snippet.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- Sections mirroring the questionnaire: Purpose · Data · Decisions · Failure · Oversight · Operations · Compliance.
- Each answer rendered as a `matrix-row` (yellow question label left, value right) from `risk-survey.yaml`.
- Values matching `TO_BE_COMPLETED_BY_DEPLOYER` render in muted italic ("To be completed by deployer").

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`. Subtitle: `Each row is one governance control. Click for rationale and implementation.`
- 5-column table: `ID | Control (obligation) | Mechanism | Implementation | Source`.
- ID badges carry a colored mechanism pill: guardrail = red, sanitizer = green, ci-gate = pale yellow.
- Rows expand vertically on click; one open at a time. The expanded row shows rationale, implementation, and `regulation_anchors`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a role. <span class="accent">Watch a posting clear.</span>` Subtitle: `The simulator also drips a request every 60 s so the page is never empty.`
- Form card: text fields labelled "Company" and "Role title", `Submit` button (yellow).
- Live list: cards per posting; left border coloured by status (PLANNING = muted, ANALYZING = blue, DRAFTED = yellow, SANITIZED = teal, CLEARED = green, DEGRADED = orange, BLOCKED = red).
  - Header row: company + role, status pill.
  - Click to expand: culture profile (values + tone + summary), role spec (responsibilities + qualifications + market range), draft body, sanitizer removed-terms chips, EEO statement, and any `failureReason`.
