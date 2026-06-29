# Akka Sample: Fun Facts

A single fact-generator agent receives a topic string and returns a structured collection of interesting facts: a headline fact, a supporting set of related facts, and a confidence tier for each. The topic rides into the agent as task instructions — no inline prompt injection.

Demonstrates the **single-agent** coordination pattern in a general-purpose domain with no governance controls: one `FactGeneratorAgent` (AutonomousAgent) drives every response; the surrounding components handle request routing, lifecycle tracking, and read-model projection. There are no guardrails or sanitizers in this baseline — deployers who need them can add controls from the eval-matrix extensions listed in `eval-matrix.yaml`.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — topics are submitted through the UI, and the agent's responses are stored in-process.

## Generate the system

```sh
cp -r ./single-agent.general.fun-facts  ~/my-projects/fun-facts
cd ~/my-projects/fun-facts
```

(Optional) Edit `SPEC.md` to change the seeded topic list or add a topic-category taxonomy for grouping facts by subject area.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FactGeneratorAgent** — an AutonomousAgent that accepts a topic and returns a typed `FactCollection`.
- **FactRequestWorkflow** — orchestrates request-intake → generation per submitted topic.
- **FactRequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **FactRequestView + FactEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded topic list for your own domain (the JSONL file under `src/main/resources/sample-events/topics.jsonl` after generation).
- `SPEC.md §5` — extend `FactCollection` with domain-specific fields (e.g., `sourceHints`, `category`, `geographicScope`).
- `prompts/fact-generator.md` — narrow the agent's role (a science publisher would constrain it to peer-reviewed-adjacent fact generation; an education platform would constrain reading level and age appropriateness).
- `eval-matrix.yaml` — wire governance controls if your deployment requires them (source attribution, output-length guardrail, content-safety filter).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a topic → the agent generates facts → the collection appears in the UI with status `GENERATED`.
2. Every `FactCollection` carries at least one fact with a non-empty `statement` and a `confidence` value.
3. Submitting the same topic twice produces two independent requests with separate `requestId` values and independent fact collections.
4. The live list updates via SSE without a page reload.

## License

Apache 2.0.
