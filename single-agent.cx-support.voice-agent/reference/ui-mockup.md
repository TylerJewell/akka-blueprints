# UI mockup — voice-agent

Five-tab structure inherited from the formal exemplar. Browser title: `<title>Akka Sample: Voice Agent</title>`.

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

The DOM contains **exactly five** `<section class="tab-panel">` elements — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more. Any panel removed in an earlier iteration must be deleted from the HTML; `display:none` is not enough. See `AKKA-EXEMPLAR-LESSONS.md` Lesson 26 for the failure mode this prevents.

## Tab 1 — Overview

- Eyebrow: `Overview`.
- Headline: `Voice<span class="accent">Agent</span>`. **No subtitle.**
- Card **Try it**: a single `/akka:build` block — no env-var export instructions. Then 4 numbered steps:
  1. Open the App UI tab.
  2. Pick a seeded scenario (Billing Query / Technical Support / Cancellation Request) and click **Load seeded example** to fill the transcript field, or type a custom transcript directly.
  3. Click **Send audio**.
  4. Watch the turn card transition through TRANSCRIPT_SANITIZED → RESPONDING → REPLY_RECORDED → AUDIO_SYNTHESISED. Click "Play audio" when it appears.
- Card **How it works**: one paragraph on receive → sanitize → respond → synthesise; one paragraph on the two governance mechanisms (PII sanitizer, before-agent-response guardrail).
- Card **Components**: rows per component (ConversationEntity, TranscriptSanitizer, ConversationWorkflow, VoiceConversationAgent, ReplyGuardrail, AudioSynthesiser, ConversationView, ConversationEndpoint, AppEndpoint) with Kind column coloured by Akka primitive type.
- Card **API contract**: rows from `api-contract.md`.

## Tab 2 — Architecture

- Eyebrow: `Architecture`. Headline: `What gets <span class="accent">wired together</span>`. Subtitle: `Component graph, sequence, state machine, entity model — then per-component detail below.`
- Stat tiles: counts per kind (1 AutonomousAgent, 1 Workflow, 1 EventSourcedEntity, 1 View, 1 Consumer, 2 HttpEndpoints, plus 1 Guardrail + 1 AudioSynthesiser as supporting classes).
- Four mermaid cards (component graph, sequence, state machine, entity model) — render with the Lesson 24 CSS overrides and `themeVariables` block. Without them, state labels render black-on-black and edge labels clip.
- Compressed comp-row table with the Java file path per component (matches `PLAN.md` Component table).

## Tab 3 — Risk Survey

- Eyebrow: `Risk Survey`. Headline: `What this <span class="accent">deployer would declare</span>`.
- 7 sub-tabs from `governance.html` rendering answers from `risk-survey.yaml`. The `pii: true` declaration in Data is filled and prominent. `decisions.authority_level = autonomous-action` and `oversight.human_on_loop = true` are the distinctive answers for a voice agent that speaks directly to callers. Many other fields are `TO_BE_COMPLETED_BY_DEPLOYER` and faded in muted italic.

## Tab 4 — Eval Matrix

- Eyebrow: `Eval Matrix`. Headline: `Controls the <span class="accent">runtime enforces</span>`.
- 5-column table with two rows (G1, S1). ID badges coloured: G1 red (guardrail), S1 green (sanitizer).
- Each row is click-to-expand; expanded view shows the full `rationale` and `implementation` paragraphs from `eval-matrix.yaml`.

## Tab 5 — App UI

- Eyebrow: `App UI`. Headline: `Speak. <span class="accent">Hear back.</span>`. Subtitle: `One agent, two governance mechanisms around it.`
- Layout: two-column.
  - **Left column** — Session panel + live list.
    - Input panel: dropdown `Scenario` (Billing Query / Technical Support / Cancellation Request / custom), `Caller ID` text input, `Transcript` textarea (with a "Load seeded example" link that fills both caller id and transcript), and a yellow `Send audio` button. When a session is active, a secondary `Add turn` button appears beneath the live list.
    - Live list below: one card per session, newest-first. Each card shows session status pill, caller id, turn count, and age.
  - **Right column** — Selected-session turn timeline.
    - Session header: session status pill + caller id + session start time.
    - Timeline: one turn block per turn in chronological order.
      - Each turn block contains:
        - Turn status pill.
        - Sanitized transcript (monospace block) with PII category chips above it.
        - Reply section: reply text, voice-tone badge (WARM / NEUTRAL / FIRM), topic classification chip, escalation flag badge (only shown when `true`).
        - Audio section: "Play audio" button (active once `AUDIO_SYNTHESISED`; stub returns a short WAV).
      - Turn blocks where status is `FAILED` show a red border and the failure reason.
- Status pill colours: AUDIO_RECEIVED=muted, TRANSCRIPT_SANITIZED=blue, RESPONDING=yellow, REPLY_RECORDED=blue, AUDIO_SYNTHESISED=green, FAILED=red.
- Voice-tone badge colours: WARM=orange, NEUTRAL=muted, FIRM=blue.
- Session status colours: ACTIVE=green, CLOSED=muted, FAILED=red.

The raw transcript is never displayed on this screen — only the sanitized form. Supervisors who need the raw text fetch `/api/conversations/{id}` and read `turns[i].request.rawTranscript` from the JSON. This is intentional: the UI demonstrates that the model's input is the redacted form, even though the audit trail keeps the raw.
