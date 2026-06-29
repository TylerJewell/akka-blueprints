# UI mockup — gitwiki

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: GitWiki Demo Agent</title>`.

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
- Headline: `Git<span class="accent">Wiki</span> Demo Agent`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded wiki page from the dropdown and type a change description.
  3. Click **Submit update**.
  4. Watch the card transition through EDITING → COMMIT_READY → PUSH_IN_PROGRESS → PUSHED.
- Card **How it works**: one paragraph on submit → config check → agent edit → push → SSE; one paragraph on the two governance mechanisms (before-tool-call guardrail and startup configuration gate).
- Card **Components**: rows per component (PageEntity, GitPushConsumer, WikiUpdateWorkflow, WikiEditorAgent, GitPushGuardrail, StartupConfigGate, PageView, WikiEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 ConfigGate as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs rendering answers from `risk-survey.yaml`. The `decisions.authority_level = autonomous` declaration is prominent. `oversight.human_on_loop = true` and `external_tool_calls: [llm-completion, github-api-commit, github-api-ref-update]` are distinctive answers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, H1). ID badges coloured: G1 red (guardrail), H1 yellow (ci-gate).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a page update. <span class="accent">Watch it commit.</span>`. Subtitle: `One agent, one guardrail, one config gate.`
- Layout: two-column.
  - **Left column** — Submission form + live list.
    - Submission form: dropdown `Page` (getting-started.md / architecture-overview.md / api-reference.md / custom), `Requested changes` textarea (with placeholder "Describe what to change — e.g. 'Add a Prerequisites section listing Java 21'"), `Author` text input, and a yellow `Submit update` button.
    - Live list below: one card per update, newest-first. Each card shows status pill, commit SHA chip (when pushed), page path, age.
  - **Right column** — Selected-update detail.
    - Header: status pill + commit SHA chip (linked to GitHub) + `pageTitle`.
    - Requested changes: the original `requestedChanges` text, faded italic.
    - Diff preview: a monospace block rendering the `diffSummary` and a two-column before/after of the changed lines (simple line diff, not a full unified diff).
    - Commit message: the agent's commit message in a monospace pill.
    - Push result section: for `PUSHED` — commit SHA, link to GitHub, `pushedAt`. For `PUSH_REJECTED` — rejection reason in a red callout. For `CONFLICT` — conflicting SHA and a "rebase manually" note.
- Status pill colours: SUBMITTED=muted, EDITING=yellow, COMMIT_READY=blue, PUSH_IN_PROGRESS=yellow, PUSHED=green, PUSH_REJECTED=red, CONFLICT=orange, FAILED=red.
- Push status badge colours: SUCCESS=green, REJECTED=red, CONFLICT=orange, FAILED=red.

The current page body is not displayed on this screen — only the diff summary and the edited body side of the preview. Users who need the current raw content fetch `/api/updates/{id}` directly. This is intentional: the UI surfaces the agent's decision, not the full document corpus.
