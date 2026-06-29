# UI mockup — cyber-guardian-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Cyber Guardian Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See AKKA-EXEMPLAR-LESSONS.md Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Cyber Guardian <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: `/akka:build` (Claude Code) block, then 4 numbered steps (open App UI, wait for the simulator to drop a signal, observe the automatic halt on CRITICAL, click Lift Halt or review the playbook).
- Card **How it works**: one paragraph on the poll → classify → halt → remediate → close → eval-event flow; one paragraph on the two governance mechanisms.
- Card **Components**: rows per component (poller, queue, consumer, classifier, advisor, halter, workflow, entity, view, eval-emitter, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 Agent, 2 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 1 Consumer, 2 TimedActions, 2 HttpEndpoints).
- Four mermaid cards.
- Compressed comp-row table with syntax-highlighted Java snippets on expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`. The `ip-addresses-and-hostnames: true` and `security-credentials: true` declarations in Data are the most distinctive — those chips are filled. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (E1, H1). ID badges coloured: E1 green (eval-event), H1 red (halt).

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the threat feed. <span class="accent">Halts fire automatically.</span>`. Subtitle: `Simulated signals arrive every 10 s. CRITICAL incidents are isolated before you read the alert.`
- Layout: two-column.
  - **Left column** — Live incident list, sorted newest-first. Each card shows:
    - Header: severity pill, status indicator, signal age.
    - Source host (monospace).
    - Category chip.
    - Halt badge (visible only when `haltIssued = true`).
  - **Right column** — the selected incident detail.
    - Assessment reasoning (read-only).
    - Confidence score bar.
    - Halt directive box (visible when `halt` is present): isolatedHost, justification, issuedAt.
    - Lift Halt button (red, visible only when `status = HALTED`) — opens a small reason textarea before confirming.
    - Playbook steps list (numbered, visible when `playbook` is present).
    - Eval event payload (small chip row, visible when `evalEvent` is present).
- Severity pill colours: CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=muted.
- Status indicator colours: RECEIVED=muted, CLASSIFIED=blue, HALTED=red, REMEDIATION=orange, CLOSED=green, IGNORED=muted.

The Lift Halt button is the only action available in this UI — the system acts automatically; the human decides when to resume.
