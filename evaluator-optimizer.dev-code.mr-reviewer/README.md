# Akka Sample: GitLab MR Reviewer

A reviewer agent reads a GitLab merge request diff; a gating agent decides whether the review commentary meets quality bar; the two iterate until the gate accepts or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance: a secret-scanning sanitizer blocks sensitive diffs before any LLM call, an output guardrail prevents review comments from reproducing proprietary code or secrets, and a CI gate publishes a pass/fail signal back to the originating merge request.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint models the GitLab webhook payload and the outbound comment API internally; no live GitLab instance is required to generate and run the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.dev-code.mr-reviewer  ~/my-projects/mr-reviewer
cd ~/my-projects/mr-reviewer
```

(Optional) Edit `SPEC.md` to change the review rubric, the maximum number of refinement passes, the secret-scanning patterns, or the CI gate threshold score.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReviewerAgent** — AutonomousAgent that reads a diff and produces structured review commentary: per-file findings, a summary, and a numeric quality score.
- **GateAgent** — AutonomousAgent that scores whether a set of review comments is thorough and actionable, returns either `PASS` or `REFINE` with typed `GateFeedback`.
- **ReviewWorkflow** — Workflow that runs the diff → sanitize → review → gate → refine loop up to a configurable retry ceiling, publishes a `CI_PASS` or `CI_FAIL` signal when done.
- **MrEntity** — EventSourcedEntity that holds the MR lifecycle, every review pass, every gate verdict, and the final CI signal.
- **MrQueue** — EventSourcedEntity that logs each incoming webhook for replay and audit.
- **MrView** — read-side projection that the UI lists and streams via SSE.
- **WebhookConsumer** — Consumer that starts a workflow per inbound MR event.
- **MrSimulator** — TimedAction that drips a sample MR webhook every 90 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records per-pass eval events each cycle (control E1).
- **ReviewEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample diffs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `MergeRequest` record fields (e.g., raise `maxPasses`, change the default CI gate threshold score).
- `prompts/reviewer.md` — narrow the review rubric (e.g., restrict to security findings only).
- `prompts/gate.md` — change the acceptance criteria (e.g., require explicit test-coverage comments).
- `eval-matrix.yaml` — tighten enforcement on the secret sanitizer (currently system-level).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an MR webhook → MR progresses `RECEIVED` → `REVIEWING` → `GATE_CHECKING` → `CI_PASS` within the retry ceiling.
2. Force-fail gate → MR hits `CI_FAIL` after the configured number of passes; the entity preserves every review pass and every gate verdict for audit.
3. The secret sanitizer blocks a diff containing a credential pattern before the reviewer agent runs; the MR transitions to `SANITIZER_BLOCKED` and no LLM call is made.
4. The CI gate output guardrail prevents review commentary that reproduces secrets or verbatim proprietary code from reaching the GitLab comment API.
5. Each completed cycle emits a `ReviewEvalRecorded` event that surfaces in the App UI's per-pass timeline.

## License

Apache 2.0.
