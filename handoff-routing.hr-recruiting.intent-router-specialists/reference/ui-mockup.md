# UI mockup — core-semantic-router

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Core Semantic Router</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Core Semantic <span class="accent">Router</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first query, click any query to inspect, notice that BLOCKED queries show the rejection reasons with no specialist answer).
- Card **How it works**: one paragraph on the sanitize → classify → guardrail → route → answer flow; one paragraph on the three governance mechanisms (PII sanitizer, before-agent-invocation guardrail, on-decision eval).
- Card **Components**: rows per component (simulator, queue, sanitizer, intent router, routing guardrail, HR specialist, finance specialist, routing judge, workflow, entity, view, eval scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24:
  - `%%{init: {'theme':'base','themeVariables':{...,'stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%`
  - `<style> .mermaid foreignObject { overflow: visible; } </style>`
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from the governance style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.employee-financial-records = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, and `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">See the guardrail work.</span>`. Subtitle: `Simulated queries drop every 30 s. The router picks the specialist; the guardrail blocks bad routes before any specialist runs.`
- Layout: three-column.
  - **Left column** — Live query list, sorted newest-first. Each card shows:
    - Header: status pill, domain chip (HR yellow, FINANCE cyan, AMBIGUOUS muted), query age, routing score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (the user never sees the raw subject).
    - PII categories found (small muted chips).
  - **Centre column** — Selected query: redacted subject + body (read-only), the routing block (domain badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's answer:
    - Specialist tag (hr yellow, finance cyan).
    - Answer body.
    - Action chip (`POLICY_CITED`, `PROCESS_EXPLAINED`, `REFERRED`, `ESCALATED`).
    - Guardrail block: green check + "authorized" when verdict passed; red badge + rejections list when blocked.
    - When `status = BLOCKED`: a muted "Blocked — no specialist invoked" block with the rejections list and the guardrail rubric version.
    - When `status = ANSWERED`: the published answer carries a green "Answered" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — domain ambiguous" block in place of the answer, with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `CLASSIFIED` blue, `AUTHORIZED` teal, `ROUTED_HR` yellow, `ROUTED_FINANCE` cyan, `ANSWERED` green, `BLOCKED` red, `ESCALATED` amber.

The routing score chip is the most distinctive surface — a continuous, per-query number visible at-a-glance in the list and broken down in the centre column. It is the on-decision eval mechanism made tangible. The guardrail block in the right column illustrates the before-agent-invocation pattern: a blocked entry shows zero specialist output because the specialist was never called.
