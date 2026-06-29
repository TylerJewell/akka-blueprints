# UI mockup — finance-document-triage

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Finance Document Triage</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not sufficient. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Finance Document <span class="accent">Triage</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block (Claude Code slash command — no env-var export, the key was handled during `/akka:specify` per Lesson 25), then four numbered steps (open App UI, wait 30 s for the simulator to drop the first document, click any document to inspect, optionally click Unblock on a `BLOCKED` document).
- Card **How it works**: one paragraph on the dual-sanitization → classify → handoff → guardrail → publish flow; one paragraph on the four governance mechanisms.
- Card **Components**: rows per component (simulator, queue, PII sanitizer, sector sanitizer, classifier agent, invoice processor, loan processor, compliance processor, classification judge, output guardrail, workflow, entity, view, scorer, endpoints) with Kind column coloured.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (3 Agents typed, 3 AutonomousAgents, 1 Workflow, 2 ESEs, 1 View, 3 Consumers, 1 TimedAction, 2 HttpEndpoints).
- Four mermaid diagram cards. Each card includes the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels do not clip at the top.
- Compressed component-row table below the diagrams with syntax-highlighted Java snippets on row expand.

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendered from `risk-survey.yaml`. The distinctive declarations are `data.pii = true`, `data.financial-data = true`, `data.pii_handled_by_sanitizer_before_llm = true`, `data.sector_data_redacted_before_llm = true`, `decisions.authority_level = draft-only`, and `oversight.human_in_loop = false` paired with `oversight.reviewer_must_unblock_guardrail_failures = true`. Most jurisdictional and operational fields are `TO_BE_COMPLETED_BY_DEPLOYER` and rendered muted-italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- Four rows: S1 (sanitizer · cyan badge), S2 (sanitizer · teal badge), G1 (guardrail · yellow badge), E1 (eval-event · blue badge). Each row expands to show the rationale + implementation paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Watch the triage. <span class="accent">Catch the violations.</span>`. Subtitle: `Simulated documents drop every 30 s. The classifier picks the processor; the guardrail catches policy breaches.`
- Layout: three-column.
  - **Left column** — Live document list, sorted newest-first. Each card shows:
    - Header: status pill, type chip (INVOICE yellow, LOAN_APPLICATION cyan, COMPLIANCE_REVIEW amber), document age, classification score chip (1–5; colour-graded — 1–2 red, 3 amber, 4–5 green).
    - Redacted subject (the user never sees the raw subject).
    - PII categories found and sector fields redacted (small muted chips in two groups).
  - **Centre column** — Selected document: redacted subject + content (read-only), the classification block (type badge + confidence + reason), and the classification score block (number + rationale + scoredAt).
  - **Right column** — The chosen processor's draft:
    - Processor tag (invoice yellow, loan cyan, compliance amber).
    - Result summary + detail.
    - Action chip (`APPROVED`, `FLAGGED_FOR_REVIEW`, `INFORMATION_REQUESTED`, `REJECTED`, `FORWARDED_TO_COMPLIANCE`, `ESCALATED`).
    - Guardrail block: green check + "allowed" when verdict allowed; red badge + violations list when blocked.
    - When `status = BLOCKED`: an Unblock button (yellow) opens a small note textarea and posts to `/api/documents/{id}/unblock`.
    - When `status = PROCESSED`: the published result carries a green "Published" stamp with `finishedAt`.
    - When `status = ESCALATED`: a muted "Escalated — workflow error or recovery failed" block in place of the draft, with the `escalationReason`.
- Status pill colours: `RECEIVED` muted, `PII_SANITIZED` / `SECTOR_SANITIZED` orange, `CLASSIFIED` blue, `ROUTED_INVOICE` yellow, `ROUTED_LOAN` cyan, `ROUTED_COMPLIANCE` amber, `RESULT_DRAFTED` purple, `BLOCKED` red, `PROCESSED` green, `ESCALATED` deep-amber.

The classification score chip is the on-decision eval mechanism made tangible — a continuous per-document signal visible at-a-glance in the list and broken down in the centre column.
