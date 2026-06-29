# UI mockup — triage-expert-multi-agent-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Triage + Expert Multi-Agent Workflow</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Triage + Expert <span class="accent">Multi-Agent Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded issue categories (or describe your own issue) and click **Submit issue**.
  3. Watch the card transition through GATHERING → GATHERED → SUMMARIZING → SUMMARIZED → SANITIZING → SANITIZED → RECOMMENDING → RECOMMENDED → EVALUATED.
  4. Inspect the guardrail-block log strip on the card if any content blocks fired.
- Card **How it works**: one paragraph on the two-agent pipeline (TriageAgent gathers and summarizes, ExpertAgent composes a knowledge-grounded recommendation) and the PII sanitizer that runs between them; one paragraph on the three governance mechanisms (response guardrail, PII sanitizer, grounding eval).
- Card **Components**: rows per component (SupportCaseEntity, SupportCaseWorkflow, TriageAgent, ExpertAgent, IntakeTools, KnowledgeBaseTools, PiiSanitizer, RecommendationGuardrail, RecommendationScorer, SupportCaseView, SupportCaseEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 AutonomousAgents, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 2 function-tool classes, 1 Sanitizer, 1 Guardrail, 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and highlighted (customer name, email, account ID, phone are collected during triage). `decisions.authority_level = recommend-only` and `oversight.human_in_loop = true` are the distinctive answers. `agent_count: 2` and `agent_pattern: sequential-pipeline` are pre-filled. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and shown in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (G1, S1, E1). ID badges coloured: G1 red (guardrail), S1 purple (sanitizer), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit an issue. <span class="accent">Get a grounded recommendation.</span>`. Subtitle: `Two agents, one sanitizer, one guardrail between them.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: textarea `Issue description` (with a "Pick a seeded issue" dropdown that fills it), and a yellow `Submit issue` button.
    - Live list below: one card per case, newest-first. Each card shows status pill, issue category chip, eval score chip (when eval landed), age, and a small red dot if any guardrail block fired during this case.
  - **Right column** — Selected-case detail.
    - Header: status pill + eval score chip + issue category chip.
    - Phase panel 1 (Customer info): table with name, email, account ID, phone, urgency, category. Labelled `[pre-sanitization]`. Visible once `customerInfo` is present. Muted styling to indicate this data does not cross the agent boundary.
    - Phase panel 2 (Triage summary): problem statement and key facts list. Visible once `issueSummary` is present.
    - Phase panel 3 (Sanitized summary): problem statement and key facts with PII placeholders highlighted in amber. A `PII scrub audit` strip shows the `redactedTokens` list. Visible once `sanitizedSummary` is present.
    - Phase panel 4 (Recommendation): guidance text, cited articles list (each with article ID, title, relevance score chip), and a `Requires escalation` badge if `requiresHumanEscalation` is true. Visible once `recommendation` is present.
    - Eval section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Guardrail-block log strip (only visible if the case has any `guardrailBlocks`): a small table with rule, excerpt, time.
- Status pill colours: CREATED=muted, GATHERING=blue, GATHERED=blue, SUMMARIZING=yellow, SUMMARIZED=yellow, SANITIZING=purple, SANITIZED=purple, RECOMMENDING=blue, RECOMMENDED=blue, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A case in `RECOMMENDING` shows panels 1–3 (panel 4 with a spinner if the expert agent has not yet returned). This is the visual proof that the sanitized summary — not the raw customer info — is the only data that crossed into the expert phase.
