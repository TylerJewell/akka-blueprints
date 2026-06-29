# UI mockup — multi-agent-handoff-workflow

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Multi-Agent Handoff Workflow</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents (clicking a tab and seeing a blank because a hidden zombie panel occupied the index).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Multi-Agent <span class="accent">Handoff Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first task, click any task to inspect, observe the routing score appear within ~10 s).
- Card **How it works**: one paragraph on the admit → route → handoff → execute → validate → publish flow; one paragraph on the three governance mechanisms.
- Card **Components**: rows per component (simulator, queue, admission guardrail, router agent, data analyst, content writer, code reviewer, routing judge, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (2 Agents typed, 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = false`, `decisions.authority_level = autonomous-with-guardrail`, `oversight.human_in_loop = false` paired with `oversight.reviewer_must_review_tool_blocked_tasks = true`, and `admission_checked_before_llm = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: G1 (guardrail · before-agent-invocation · yellow badge), G2 (guardrail · before-tool-call · orange badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the handoff. <span class="accent">Catch the tool violations.</span>`. Subtitle: `Simulated tasks drop every 30 s. The router picks the specialist; the guardrails catch the misses.`
- Layout: three-column.
  - **Left column** — Live task list, sorted newest-first. Each card shows:
    - Header: status pill, domain chip (`DATA_ANALYSIS` cyan, `CONTENT_WRITING` purple, `CODE_REVIEW` yellow, `UNROUTABLE` muted), task age, routing-score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Task title.
    - Result format chip when present (small muted indicator).
  - **Centre column** — Selected task: title + description (read-only), the routing block (domain badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The chosen specialist's result:
    - Specialist tag (data-analyst cyan, content-writer purple, code-reviewer yellow).
    - Result headline + body (rendered as markdown when `format = MARKDOWN_REPORT`).
    - Format chip (`MARKDOWN_REPORT`, `STRUCTURED_JSON`, `INLINE_COMMENT`, `PLAIN_TEXT`).
    - Tool-call record block when present: tool name + block reason when `allowed=false` (red badge); checkmark when `allowed=true`.
    - When `status = TOOL_BLOCKED`: a red "Tool blocked — operator review required" banner with the blocked tool name and block reason. No publish button.
    - When `status = COMPLETED`: the result carries a green "Published" stamp with `finishedAt`.
    - When `status = REJECTED`: a muted "Rejected — no specialist invoked" block in place of the result, with the `rejectionReason`.
    - When `status = VALIDATION_FAILED`: a yellow "Validation failed" block with the validation message.
- Status pill colours: `RECEIVED` muted, `ADMITTED` blue, `ROUTED` indigo, `ROUTED_DATA_ANALYSIS` cyan, `ROUTED_CONTENT_WRITING` purple, `ROUTED_CODE_REVIEW` yellow, `EXECUTING` amber, `TOOL_BLOCKED` red, `VALIDATION_FAILED` orange, `COMPLETED` green, `REJECTED` muted-red.

The routing-score chip is the most distinctive surface — a continuous, per-task number visible at-a-glance in the list and broken down in the centre column. It is the on-decision eval mechanism made tangible.
