# UI mockup — global-kyc-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Global KYC Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough (Lesson 26).

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Global KYC <span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick one of the seeded applicant profiles (or fill in the form fields) and click **Start KYC**.
  3. Watch the card transition through COLLECTING → VERIFYING → DECIDING → DECIDED → EVALUATED (or pause at PENDING_REVIEW if the outcome is DECLINE).
  4. For PENDING_REVIEW cases, click **Approve** or **Override to Pass** in the review panel.
- Card **How it works**: one paragraph on the three task phases (COLLECT → VERIFY → DECIDE) and the typed handoff between them; one paragraph on the three governance mechanisms (PII sanitizer, HITL review gate, decision completeness eval).
- Card **Components**: rows per component (KycCaseEntity, KycPipelineWorkflow, KycAgent, CollectTools, VerifyTools, DecideTools, PiiSanitizer, HitlReviewGate, DecisionScorer, KycView, KycEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 2 HttpEndpoints, plus 3 function-tool classes, 1 Sanitizer, 1 HITL gate, and 1 Scorer as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled (applicant identity documents contain full legal name, date of birth, and document numbers). `decisions.authority_level = automated-with-human-override` and `oversight.human_in_loop = true` are the distinctive answers. `oversight.adverse_outcome_requires_sign_off = true` is also filled and highlighted. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with three rows (S1, H1, E1). ID badge colours: S1 amber (sanitizer), H1 cyan (hitl), E1 blue (eval-event).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Submit a profile. <span class="accent">Get a KYC decision.</span>`. Subtitle: `One agent, three task phases, three runtime governance layers.`
- Layout: two-column.
  - **Left column** — Submission panel + live list.
    - Submission panel: a "Pick a seeded profile" dropdown that fills the form, plus individual fields (Applicant ID, Jurisdiction, Document Types checkboxes) and a teal **Start KYC** button.
    - Live list below: one card per case, newest-first. Each card shows status pill, outcome chip (when decided), applicant ID, age, and a yellow caution dot if the eval score is ≤ 2.
  - **Right column** — Selected-case detail.
    - Header: status pill + outcome chip + applicant ID + jurisdiction.
    - Phase panel 1 (Collected documents): a table with columns document type, issuing jurisdiction, tokenised name, tokenised DOB, tokenised doc number, fetch status. Visible once `documents` is present.
    - Phase panel 2 (Verification): a document-status list (document ID → status badge → status reason) and a rule-check list (rule ID → passed/failed badge → failure reason). Visible once `verification` is present.
    - Phase panel 3 (Decision): outcome badge (PASS=green, DECLINE=red, REFER=amber, PENDING_DOCUMENTS=yellow), jurisdiction, cited rule IDs as chips, rationale text. Visible once `decision` is present.
    - HITL review panel (only visible when `status == PENDING_REVIEW`): shows the decision card and a two-button bar — **Approve as submitted** and **Override to Pass** — each prompting for a notes field before posting to `/api/cases/{id}/review`.
    - Eval section at bottom: a 1–5 star widget and the one-line scorer rationale. Score ≤ 2 highlights the card border yellow.
    - Review record strip (visible when `review` is present): reviewer ID, resolution, notes, resolved timestamp.
- Status pill colours: CREATED=muted, COLLECTING=blue, COLLECTED=blue, VERIFYING=yellow, VERIFIED=yellow, DECIDING=blue, DECIDED=green, PENDING_REVIEW=amber, REVIEWED=amber, EVALUATED=green, FAILED=red.

Each phase panel renders only when its data is present on the row record. A case in `VERIFYING` shows panel 1 (documents, tokenised) and panel 2 with a "verifying…" spinner. This is the visual proof that the typed handoff between phases is the only path information travels — and that PII fields are always tokens in the UI regardless of pipeline stage.
