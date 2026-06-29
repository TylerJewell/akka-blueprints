# Akka Sample: On-Brand GenMedia

A brand-agent generates visual-ad copy and social captions for a given campaign brief; a brand-reviewer agent scores the output against a configurable brand rubric; the two iterate until the reviewer approves the asset or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with an after-response brand/safety guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the campaign request stream and the brand-approval surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.sales-marketing.on-brand-media  ~/my-projects/on-brand-genmedia
cd ~/my-projects/on-brand-genmedia
```

(Optional) Edit `SPEC.md` to change the brand voice rules, the reviewer rubric, the token ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BrandAgent** — AutonomousAgent that generates ad copy and social captions on a campaign brief, incorporating prior reviewer feedback on revision calls.
- **ReviewerAgent** — AutonomousAgent that scores a media asset against the brand rubric, returning either `APPROVE` or `REVISE` with a typed `ReviewNotes` payload.
- **GenerationWorkflow** — Workflow that runs the generate → guardrail → review → revise loop up to a configurable retry ceiling, transitioning the asset to `APPROVED` on reviewer approval or to `REJECTED_FINAL` when the ceiling is reached.
- **AssetEntity** — EventSourcedEntity holding the asset lifecycle, every generation attempt, every review, and the final outcome.
- **CampaignQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **AssetsView** — read-side projection that the UI lists and streams via SSE.
- **CampaignRequestConsumer** — Consumer that starts a workflow per inbound submission.
- **CampaignSimulator** — TimedAction that drips a sample campaign brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **BrandEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the campaign briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Asset` record fields (e.g., raise `maxAttempts`, change the token ceiling).
- `prompts/brand-agent.md` — narrow the voice to a specific brand persona or channel (e.g., LinkedIn B2B, Instagram consumer).
- `prompts/reviewer-agent.md` — change the rubric (e.g., add a tone-of-voice constraint or a regulatory-language check).
- `eval-matrix.yaml` — tighten enforcement on the brand/safety guardrail.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a campaign brief → asset progresses `GENERATING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → asset hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every review for audit.
3. The brand/safety guardrail blocks an asset containing prohibited language before the reviewer sees it, so the brand agent re-generates a clean version.
4. Each completed review cycle emits a `ReviewEvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
