# UI mockup — ig-dm-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: IG DM Agent</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The exemplar's `static-resources/index.html` has the canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `IG DM <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded brand profile (Retail / Hospitality / SaaS) and load the matching seed DM, or paste your own.
  3. Click **Send to agent**.
  4. Watch the card transition through SANITIZED → REPLYING → REPLIED.
- Card **How it works**: one paragraph on receive → sanitize → reply; one paragraph on the two governance mechanisms (PII sanitizer, brand-safety guardrail).
- Card **Components**: rows per component (DmEntity, MessageSanitizer, DmWorkflow, DmReplyAgent, ReplyGuardrail, DmView, DmEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail as supporting class).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous` and `oversight.human_on_loop = true` are the distinctive answers for a CX support automation. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a DM. <span class="accent">Get the reply.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: dropdown `Brand profile` (Retail / Hospitality / SaaS / custom), `Sender ID` text input, `Message` textarea (with a "Load seeded example" link that fills both sender and message body), and a yellow `Send to agent` button.
    - Live list below: one card per message, newest-first. Each card shows status pill, tone label chip (when reply landed), document title, age.
  - **Right column** — Selected-message detail.
    - Header: status pill + tone label chip + `senderId`.
    - Raw DM preview: the original message text with PII category chips above (`email`, `phone`, etc.).
    - Sanitized message preview: a monospace block of the redacted text.
    - Draft reply section: the `replyText` in a reply bubble style, tone label, and a 1–5 star widget for `toneConfidenceScore`.
    - Brand profile summary: `profileId`, `maxReplyChars`, count of `prohibitedTerms`.
- Status pill colours: RECEIVED=muted, SANITIZED=blue, REPLYING=yellow, REPLIED=green, FAILED=red.
- Tone label chip colours: FRIENDLY=green, PROFESSIONAL=blue, EMPATHETIC=purple, NEUTRAL=muted.

The raw DM is visible in the right pane (read-only) to help moderators audit the interaction. The sanitized form is shown separately so the reviewer can confirm what the model actually saw.
