# UI mockup — cx-handoff-triage

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Core Streaming Handoffs Customer Service</title>`.

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
- Headline: `Core Streaming Handoffs <span class="accent">Customer Service</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first conversation, click any card to inspect, optionally click Unblock on a `BLOCKED` conversation).
- Card **How it works**: one paragraph on the sanitize → triage → handoff → tool-call guardrail → response guardrail → publish flow; one paragraph on the three governance mechanisms.
- Card **Components**: rows per component (simulator, queue, sanitizer, triage agent, sales specialist, issues-repairs specialist, response guardrail, tool-call guardrail, workflow, entity, view, endpoints) with Kind column colour-coded.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, `decisions.tool_calls_validated_before_execution = true`, and `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: G1 (guardrail · yellow badge), G2 (guardrail · orange badge), S1 (sanitizer · cyan badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Catch the policy violations.</span>`. Subtitle: `Simulated conversations drop every 30 s. Triage picks the specialist; two guardrails catch the misses.`
- Layout: three-column.
  - **Left column** — Live conversation list, sorted newest-first. Each card shows:
    - Header: status pill, category chip (SALES yellow, ISSUES_REPAIRS cyan, UNCLEAR muted), conversation age.
    - Redacted opening message excerpt (the user never sees the raw text).
    - PII categories found (small muted chips).
  - **Centre column** — Selected conversation: redacted opening message (read-only) and the triage block (category badge + confidence + reason).
  - **Right column** — The chosen specialist's draft:
    - Specialist tag (sales yellow, issues-repairs cyan).
    - Response text.
    - Action chip (`ORDER_PLACED`, `REFUND_INITIATED`, `REPLACEMENT_ARRANGED`, `INFO_PROVIDED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`).
    - Tool-call block (if any `ToolCallBlocked` events exist): shows the tool name and the rejection reason, with a muted "cancelled before execution" label.
    - Guardrail block: green check + "allowed" when the response guardrail passes; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/conversations/{id}/unblock`.
    - When `status = RESOLVED`: the published response carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — no specialist invoked" block in place of the draft, with the `escalationReason`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `TRIAGED` blue, `ROUTED_SALES` yellow, `ROUTED_ISSUES_REPAIRS` cyan, `RESPONSE_DRAFTED` purple, `TOOL_CALL_BLOCKED` orange, `BLOCKED` red, `RESOLVED` green, `ESCALATED` amber.

The tool-call block in the right column is the most distinctive surface compared to a plain handoff sample — it shows the guardrail verdict for each attempted tool call, making the before-tool-call governance mechanism visible to an operator reviewing the conversation.
