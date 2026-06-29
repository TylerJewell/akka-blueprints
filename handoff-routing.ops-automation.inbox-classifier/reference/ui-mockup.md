# UI mockup — inbox-classifier

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: InboxZero Lite — Email Classifier</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel that was removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `InboxZero Lite — <span class="accent">Email Classifier</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first message, click any message to inspect, optionally click Unblock on a `BLOCKED` message).
- Card **How it works**: one paragraph on the sanitize → classify → route → guardrail → action flow; one paragraph on the two governance mechanisms (PII sanitizer and before-tool-call guardrail) and the on-decision eval.
- Card **Components**: rows per component (simulator, queue, sanitizer, classifier agent, routing agent, action guardrail, classification judge, eval scorer, workflow, entity, view, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 1 AutonomousAgent, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, `decisions.reversibility = partially-reversible`, and `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Two rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the classifier. <span class="accent">Catch the bad actions.</span>`. Subtitle: `Simulated messages drop every 30 s. The classifier picks the label; the guardrail stops the dangerous moves.`
- Layout: three-column.
  - **Left column** — Live message list, sorted newest-first. Each card shows:
    - Header: status pill, label chip (URGENT red, IMPORTANT yellow, INFO cyan, SPAM muted), message age, classification score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Redacted subject (the user never sees the raw subject).
    - PII categories found (small muted chips).
  - **Centre column** — Selected message: redacted subject + body (read-only), the classification block (label badge + confidence + reason), and the classification score block (number + rationale + scoredAt).
  - **Right column** — The routing decision:
    - Action chip (`FLAG_URGENT`, `MOVE_TO_FOLDER`, `MARK_READ`, `MOVE_TO_SPAM`, `FORWARD_TO_HUMAN`, `DELETE`).
    - Target folder (when applicable).
    - Routing reason.
    - Guardrail block (only present for destructive actions): green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/messages/{id}/unblock`.
    - When `status = ACTIONED`: a green "Actioned" stamp with `finishedAt` and the action taken.
    - When `status = ESCALATED`: a muted "Escalated — workflow error" block with the `blockReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `CLASSIFIED` blue, `ROUTING_DECIDED` purple, `GUARDRAIL_PENDING` amber, `ACTIONED` green, `BLOCKED` red, `ESCALATED` orange.

The classification score chip is the most distinctive surface — a per-message quality signal visible at-a-glance in the list and broken down in the centre column. It is the on-decision eval mechanism made tangible.
