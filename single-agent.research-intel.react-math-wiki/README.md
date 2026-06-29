# Akka Sample: Math + Wikipedia ReAct

A single research agent answers factual and numeric questions by deciding, step-by-step, which tool to call next: `evaluate_math` to compute arithmetic expressions, or `search_wikipedia` to retrieve encyclopedia passages. The agent runs a ReAct loop — Reason, then Act — and produces a typed `ResearchAnswer` once it has gathered enough evidence.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that sandboxes every `evaluate_math` call, preventing arbitrary code from executing outside the permitted expression grammar.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the Wikipedia tool is a thin in-process stub seeded with curated passages; the math evaluator is a restricted expression engine.

## Generate the system

```sh
cp -r ./single-agent.research-intel.react-math-wiki  ~/my-projects/react-math-wiki
cd ~/my-projects/react-math-wiki
```

(Optional) Edit `SPEC.md` to swap the seeded Wikipedia passage set for a different topic corpus, or narrow the math grammar to a specific operator subset.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchAgent** — an AutonomousAgent that receives a question as its task and runs the ReAct loop: reason about which tool to call, call `evaluate_math` or `search_wikipedia`, observe the result, repeat until the answer is ready.
- **ResearchWorkflow** — one workflow per question; orchestrates submit → run → score.
- **ResearchEntity** — an EventSourcedEntity holding the per-question lifecycle and the full tool-call trace.
- **MathGuardrail** — validates every `evaluate_math` argument against a restricted grammar before the tool executes, blocking forbidden constructs.
- **ResearchView + ResearchEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded question set for your own (the JSONL file under `src/main/resources/sample-events/seed-questions.jsonl` after generation).
- `SPEC.md §5` — extend `ResearchAnswer` with domain-specific fields (e.g., `confidenceLevel`, `citationUrls`, `topicCategory`).
- `prompts/research-agent.md` — narrow the agent's scope (a mathematics tutor would restrict it to step-by-step derivations; a science wiki would widen the Wikipedia corpus).
- `eval-matrix.yaml` — replace the in-process Wikipedia stub with a real API call by naming the endpoint under the tool mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a numeric word-problem → the agent calls `evaluate_math` → a typed `ResearchAnswer` appears in the UI within 30 s.
2. A user submits a factual question → the agent calls `search_wikipedia` one or more times → the answer cites the retrieved passage.
3. The `before-tool-call` guardrail blocks a math call containing a forbidden construct; the agent reformulates and retries within its iteration budget.
4. Every recorded answer carries a completeness-eval score visible on the same UI card.

## License

Apache 2.0.
