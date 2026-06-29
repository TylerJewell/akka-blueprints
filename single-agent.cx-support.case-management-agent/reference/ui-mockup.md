# UI mockup — case-management-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Case Management Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Case Management <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded message (billing dispute / technical outage / account-access request) or type your own.
  3. Click **Send message**.
  4. Watch the case transition through OPEN → IN_PROGRESS, with the agent's action and eval score appearing on the card.
- Card **How it works**: one paragraph on receive → sanitize → agent-turn → action-apply → eval; one paragraph on the three governance mechanisms (PII sanitizer, before-agent-response guardrail, before-tool-call guardrail).
- Card **Components**: rows per component (CaseEntity, MessageSanitizer, CaseWorkflow, SupportAgent, ActionGuardrail, ToolCallGuardrail, ActionEvaluator, CaseView, CaseEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 2 Guardrails + 1 Evaluator as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc). Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` and `payment-card-data: true` declarations in Data are filled and prominent. `decisions.authority_level = operational-action` and `oversight.human_on_loop = true` are the distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, G1, G2). ID badges coloured: S1 green (sanitizer), G1 red (guardrail), G2 red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Send a message. <span class="accent">Watch the case open.</span>`. Subtitle: `One agent, two guardrails, one sanitizer around it.`
- Layout: two-column.
  - **Left column** — Message submission panel + live case list.
    - Submission panel: dropdown `Seeded message` (billing dispute / technical outage / account-access request), `Customer ID` text input, `Existing case ID` text input (optional), `Message` textarea (with a "Load seeded example" link that fills all fields), channel selector (WEB_CHAT / EMAIL / PHONE_TRANSCRIPT / API), and a yellow `Send message` button.
    - Live case list below: one card per case, newest-first. Each card shows status pill, priority chip, category chip, tier badge, eval score chip, last summary, age.
  - **Right column** — Selected-case detail.
    - Header: status pill + priority chip + tier badge + case id.
    - Sanitized message: a monospace block of the redacted message, with PII category chips above (`email`, `phone`, `payment-card`, etc.).
    - Agent action panel: action type badge, category, priority, tier, summary paragraph, and agent reasoning (collapsible).
    - CRM record snapshot: case id, opened-at, resolved-at (if set), message-ids list.
    - Eval section at bottom: a 1–5 score widget and the one-line rationale. Score ≤ 2 highlights the card border red.
- Status pill colours: OPEN=muted, IN_PROGRESS=blue, ESCALATED=yellow, RESOLVED=green, FAILED=red.
- Priority chip colours: LOW=muted, MEDIUM=blue, HIGH=yellow, CRITICAL=red.
- Action type badge colours: CREATE=green, UPDATE=blue, ESCALATE=yellow, CLOSE=muted.

The raw message text is never displayed in the right-column panel — only the sanitized form. Operators who need the raw text fetch `/api/cases/{id}` and read `lastMessage.rawText` from the JSON. This demonstrates that the model's input is the redacted form, while the audit trail retains the raw.
