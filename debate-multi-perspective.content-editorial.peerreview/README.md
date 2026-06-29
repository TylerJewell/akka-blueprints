# Akka Sample: Peer Review Panel

A review moderator runs a document past three specialist reviewers — Technical, Style, and Compliance — each assessing a different axis, then reconciles their findings into one overall verdict. Demonstrates the **debate-multi-perspective** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — every external surface is modelled inside the service with Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./debate-multi-perspective.content-editorial.peerreview  ~/my-projects/peerreview
cd ~/my-projects/peerreview
```

(Optional) Edit `SPEC.md` to change the review axes, the verdict schema, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReviewModerator** — AutonomousAgent that briefs each reviewer on its axis, then reconciles the three findings into an overall verdict.
- **TechnicalReviewer** — AutonomousAgent assessing factual and technical accuracy.
- **StyleReviewer** — AutonomousAgent assessing clarity, tone, and structure.
- **ComplianceReviewer** — AutonomousAgent assessing policy and regulatory fit.
- **EvalJudge** — AutonomousAgent that scores how mutually consistent the three axis findings are.
- **ReviewWorkflow** — Workflow that redacts PII, fans the document out to the three reviewers in parallel, then calls the moderator for synthesis.
- **ReviewEntity** — EventSourcedEntity holding the full review lifecycle.
- **ReviewView** — projection used by the UI.
- **ReviewEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample documents the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AxisReview` or `OverallVerdict` record fields (e.g., add a `confidence` field).
- `prompts/compliance-reviewer.md` — narrow the compliance axis to a single framework.
- `eval-matrix.yaml` — broaden the PII sanitizer's pattern table, or add a `before-tool-call` guardrail if you wire a real publishing target.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a document → review enters `INTAKE`, then `REVIEWING`, then `SYNTHESISED` with three axis verdicts and one overall verdict.
2. PII in the submitted body is redacted before any reviewer sees it; the redaction count surfaces in the UI.
3. A reviewer timeout drives the review to `DEGRADED` with the overall verdict reconciled from whichever axes returned.
4. The output guardrail blocks a malformed or policy-violating verdict; the review enters `BLOCKED`.
5. The eval sampler scores per-axis consistency and surfaces the score on the App UI.

## License

Apache 2.0.
