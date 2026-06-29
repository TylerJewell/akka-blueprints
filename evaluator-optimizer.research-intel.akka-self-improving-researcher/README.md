# Akka Sample: Self-Improving Deep Research Agent

A research agent issues a structured query, evaluates the quality of what it retrieved, and rewrites its own system prompt and memory blocks based on the session's outcome — so each research run makes the next one more effective. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance for a self-modifying AI system.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint models the research session, the quality evaluator, and the memory updater entirely inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.research-intel.akka-self-improving-researcher  ~/my-projects/self-improving-researcher
cd ~/my-projects/self-improving-researcher
```

(Optional) Edit `SPEC.md` to change the research query topics the simulator drips, the quality rubric the evaluator applies, or the memory-block schema the prompt rewriter updates.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchAgent** — AutonomousAgent that executes a structured research query, synthesizing findings into a `ResearchReport` with cited evidence segments.
- **EvaluatorAgent** — AutonomousAgent that scores a `ResearchReport` against a quality rubric (coverage, evidence quality, synthesis coherence, source diversity), returning either `SUFFICIENT` or `REFINE` with a typed `QualityNotes` payload.
- **PromptRewriterAgent** — AutonomousAgent that reads the completed session's `ResearchReport`, the evaluator's final verdict, and the existing `PromptMemory` blocks, then produces updated `PromptMemory` reflecting lessons learned.
- **ResearchSessionWorkflow** — Workflow that runs the research → evaluate → refine loop up to a configurable attempt ceiling, then unconditionally calls the `PromptRewriterAgent` to update memory before ending.
- **SessionEntity** — EventSourcedEntity holding the session lifecycle, every research attempt, every quality evaluation, and the final prompt-memory diff.
- **QueryQueue** — EventSourcedEntity that logs each submitted research query for replay and audit.
- **SessionsView** — read-side projection that the UI lists and streams via SSE.
- **QueryConsumer** — Consumer that starts a workflow per inbound query submission.
- **QuerySimulator** — TimedAction that drips a sample research query every 90 s so the App UI is never empty.
- **DriftSampler** — TimedAction that periodically compares the current `PromptMemory` fingerprint against the baseline recorded at service start, emitting a `PromptDriftRecorded` event when a change is detected.
- **ResearchEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the query topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., raise `maxAttempts`, change the memory-block schema).
- `prompts/research-agent.md` — narrow the research strategy (e.g., restrict to primary sources only, add a citation format requirement).
- `prompts/evaluator-agent.md` — change the quality rubric (e.g., add a recency constraint, require a minimum number of distinct sources).
- `prompts/prompt-rewriter-agent.md` — change what the rewriter is allowed to update (e.g., restrict to strategy hints only, forbid changes to the role section).
- `eval-matrix.yaml` — tighten enforcement on the drift-watch control or the recertification gate.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a query → session progresses `RESEARCHING` → `EVALUATING` → `ACCEPTED` within the attempt ceiling; memory blocks are updated and the diff is visible in the App UI.
2. Force-fail rubric → session hits `MAX_ATTEMPTS_REACHED` after the configured number of attempts; memory is still updated (the rewriter sees the failure pattern); every attempt and every evaluation is preserved for audit.
3. Drift detection → after a successful session updates memory, the `DriftSampler` detects the fingerprint change and emits a `PromptDriftRecorded` event visible in the App UI's drift timeline.
4. Each completed evaluation cycle appears as a `QualityEvalRecorded` event in the per-session timeline, showing the verdict, the rubric score, and which memory blocks changed.

## License

Apache 2.0.
