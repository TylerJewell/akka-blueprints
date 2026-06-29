# UI mockup — social-amplifier

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: AI-Powered Social Media Amplifier</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index. The canonical implementation:

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `AI-Powered Social Media <span class="accent">Amplifier</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded articles (or paste your own URL) and select target platforms.
  3. Click **Amplify** and watch the card transition through PARSING → PARSED → DRAFTING → DRAFTED → PUBLISHING → PUBLISHED.
  4. Inspect the brand-check-rejection log strip on the card if any policy violations fired.
- Card **How it works**: one paragraph on the three task phases (PARSE → DRAFT → PUBLISH) and the typed handoff between them; one paragraph on the two governance mechanisms (brand-policy response guardrail, publish-intent tool-call guardrail).
- Card **Components**: rows per component (AmplificationEntity, AmplificationWorkflow, AmplifierAgent, ParseTools, DraftTools, PublishTools, BrandPolicyGuardrail, BrandPolicyScorer, AmplificationView, AmplificationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 dual-hook Guardrail, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: false` declaration in Data is filled (article text and drafts are not person-level data). `decisions.authority_level = automated-action` and `oversight.human_on_loop = true` are the distinctive answers — the marketer reviews the brand-audit score after publish, not before. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, G2). Both ID badges are red (guardrail).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Paste an article. <span class="accent">Publish to every platform.</span>`. Subtitle: `One agent, three task phases, two runtime guardrails.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: text input `Article URL` (with a "Pick a seeded article" dropdown that fills it), a platform checklist (LinkedIn, X, Bluesky — all checked by default), and a yellow `Amplify` button.
    - Live list below: one card per run, newest-first. Each card shows status pill, brand-audit score chip (when audit landed), source URL (truncated), age, and a small orange dot if any brand-check rejection fired during this run.
  - **Right column** — Selected-run detail.
    - Header: status pill + brand-audit score chip + article headline.
    - Phase panel 1 (Parsed article): article headline and a table of key messages with columns messageId, text, platformHint. Visible once `parsedArticle` is present.
    - Phase panel 2 (Drafted posts): one draft card per platform, each showing the draft text, hashtags, character count / limit bar, and a brand-check badge (green pass / orange retried / red failed). Visible once `draftSet` is present.
    - Phase panel 3 (Publication receipts): one receipt row per platform with platform name, postUrlStub link, and publishedAt timestamp. Visible once `publishedSet` is present.
    - Brand-audit section at bottom: a 1–5 star widget and the one-line rationale. Score ≤ 2 highlights the card border red.
    - Brand-check-rejection log strip (only visible if the run has any `brandCheckRejections`): a small table with draftId, platform, rule, reason, time.
    - Publish-tool-blocked log strip (only visible if the run has any blocked publish calls): a small table with platform, tool, reason, time.
- Status pill colours: CREATED=muted, PARSING=blue, PARSED=blue, DRAFTING=yellow, DRAFTED=yellow, PUBLISHING=blue, PUBLISHED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A run in `DRAFTING` shows panel 1 (completed) and panel 2 (in-progress spinner on any platform whose draft has not yet passed brand review). This is the visual proof that the typed handoff between phases is the only path information travels.
