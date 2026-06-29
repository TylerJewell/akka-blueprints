# Akka Sample: Reflexion Self-Critique

An actor agent answers research questions by grounding every claim in tool-retrieved citations; a self-critique step then scores the answer against coverage, accuracy, and citation completeness, producing a verbal reinforcement note that conditions the next attempt. The loop continues until the critique passes all dimensions or the retry ceiling is reached.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the tool-call surface and the citation retrieval are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.research-intel.reflexion-self-critique  ~/my-projects/reflexion-self-critique
cd ~/my-projects/reflexion-self-critique
```

(Optional) Edit `SPEC.md` to change the critique rubric, the per-answer citation floor, the search tool stub, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ActorAgent** — AutonomousAgent that answers a research question by issuing tool calls to retrieve source documents, then synthesizes a cited answer grounded in those results.
- **ReflexionAgent** — AutonomousAgent that scores an answer against a rubric (citation count, coverage, accuracy self-check, source quality) and produces a verbal reinforcement note, returning either `PASS` or `RETRY` with the typed `ReflexionNote` payload.
- **ReflexionWorkflow** — Workflow that runs the answer → critique → revise loop up to a configurable retry ceiling, transitions the query to `RESOLVED` on critique approval or to `EXHAUSTED` when the ceiling is hit.
- **QueryEntity** — EventSourcedEntity that holds the query lifecycle, every attempt's answer, every reflexion note, and the final outcome.
- **QueryQueue** — EventSourcedEntity that logs each inbound question for replay and audit.
- **QueriesView** — read-side projection that the UI lists and streams via SSE.
- **QueryConsumer** — Consumer that starts a workflow per inbound question.
- **QuerySimulator** — TimedAction that drips a sample question every 60 s so the App UI is never empty.
- **ReflexionSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **ResearchEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Query` record fields (e.g., raise `maxAttempts`, lower the citation floor).
- `prompts/actor.md` — narrow the research domain (e.g., medical literature, case law, financial filings).
- `prompts/reflexion.md` — change the rubric (e.g., add a recency constraint, require primary sources only).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a question → query progresses `RESEARCHING` → `REFLECTING` → `RESOLVED` within the retry ceiling; the App UI shows every attempt's answer, citation list, and reflexion note.
2. Force-fail rubric → query hits `EXHAUSTED` after the configured number of attempts; the entity preserves every attempt and every note for audit.
3. The output guardrail blocks an answer that cites fewer sources than the citation floor, so the reflexion agent never sees under-cited material.
4. Each completed cycle emits a `ReflexionRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
