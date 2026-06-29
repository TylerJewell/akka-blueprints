# UI mockup — conditional-recruiting-router

Five-tab structure. Browser title: `<title>Akka Sample: Conditional Email/Interview Workflow</title>`.

## Tab switching — MUST be attribute-based

The five `.nav-tab` elements carry `data-tab="0"` through `data-tab="4"`. The five `.tab-panel` `<section>` elements carry matching `data-panel="0"` through `data-panel="4"`. The click handler MUST switch tabs by matching these attributes — never by NodeList index.

```js
function switchTab(target) {
  const key = String(target);
  document.querySelectorAll('.nav-tab').forEach(t =>
    t.classList.toggle('active', t.dataset.tab === key));
  document.querySelectorAll('.tab-panel').forEach(p =>
    p.classList.toggle('active', p.dataset.panel === key));
}
```

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (see AKKA-EXEMPLAR-LESSONS.md Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Conditional Email/<span class="accent">Interview Workflow</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify`), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first email, click any application to inspect, optionally click Unblock on a `TOOL_BLOCKED` application).
- Card **How it works**: one paragraph on the sanitize → classify → handoff → tool-guard → complete flow; one paragraph on the two controls (PII sanitizer and before-tool-call guardrail) and the on-decision eval.
- Card **Components**: rows per component (simulator, inbox queue, sanitizer, classifier agent, info requester, interview organizer, routing judge, tool-call guardrail, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 2 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` style with answers from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `decisions.authority_level = autonomous-with-guardrail`, and `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Three rows: S1 (sanitizer · cyan badge), G1 (guardrail · yellow badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the routing. <span class="accent">Catch the bad tool calls.</span>`. Subtitle: `Simulated candidate emails drop every 30 s. The classifier picks the specialist; the guardrail catches invalid tool calls.`
- Layout: three-column.
  - **Left column** — Live application list, sorted newest-first. Each card shows:
    - Header: status pill, route chip (INFO_REQUEST cyan, INTERVIEW_REQUEST green, UNROUTABLE muted), application age, routing score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Sanitized subject (the user never sees the raw subject).
    - PII categories found (small muted chips).
  - **Centre column** — Selected application: redacted subject + body (read-only), the routing block (route badge + confidence + reason), and the routing score block (number + rationale + scoredAt).
  - **Right column** — The specialist's output:
    - **INFO_REQUEST path**: specialist tag (cyan), reply subject + body, action chip (`QUESTION_ANSWERED`, `ARTICLE_LINKED`, etc.).
    - **INTERVIEW_REQUEST path**: specialist tag (green), interview format + interviewer ID + proposed slot, outcome chip (`SLOT_BOOKED`, `CANDIDATE_DEFERRED`, `ESCALATED_TO_RECRUITER`). Tool guardrail block: green check when allowed; red badge + violations list when blocked. When `status = TOOL_BLOCKED`: an Unblock button (yellow) opens a note textarea and posts to `/api/applications/{id}/unblock`.
    - **UNROUTABLE path**: a muted "Unroutable — no specialist invoked" block with the `closureReason`.
    - When `status = COMPLETED`: a green "Completed" stamp with `finishedAt`.
- Status pill colours: `RECEIVED` / `SANITIZED` muted, `CLASSIFIED` blue, `ROUTED_INFO` cyan, `ROUTED_INTERVIEW` green, `REPLY_DRAFTED` purple, `SLOT_PROPOSED` teal, `TOOL_BLOCKED` red, `COMPLETED` green, `UNROUTABLE_CLOSED` amber.

The routing score chip is the most visible quality signal — a continuous per-application number in the list and a detailed breakdown in the centre column. It is the on-decision eval mechanism made tangible.
