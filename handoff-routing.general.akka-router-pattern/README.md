# Akka Sample: OpenAI Agents Routing Pattern

A `ClassifierAgent` examines an incoming task request and routes it to the appropriate specialist — `ContentSpecialist`, `CodeSpecialist`, or `DataSpecialist` — which owns execution end-to-end. A durable `RouterWorkflow` records both the routing decision and the downstream specialist result, and a `before-agent-invocation` guardrail checks the routed request before the specialist begins work.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound task stream and the outbound result surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.general.akka-router-pattern  ~/my-projects/akka-router-pattern
cd ~/my-projects/akka-router-pattern
```

(Optional) Edit `SPEC.md` to change the routing taxonomy (add `ResearchSpecialist`, `TranslationSpecialist`, etc.) or to adjust the pre-invocation guardrail rubric.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TaskSimulator** — TimedAction firing every 30 s that drips canned task requests from a JSONL file into `TaskQueue`.
- **TaskQueue** — EventSourcedEntity append-only log of every inbound request (audit before classification).
- **ClassifierAgent** — typed Agent that classifies the request into `CONTENT`, `CODE`, `DATA`, or `UNKNOWN`.
- **RoutingGuardrail** — typed Agent that runs a `before-agent-invocation` check on the classified request before handing off to the specialist.
- **ContentSpecialist** — AutonomousAgent that owns the `EXECUTE` task for content-domain requests.
- **CodeSpecialist** — AutonomousAgent that owns the `EXECUTE` task for code-domain requests.
- **DataSpecialist** — AutonomousAgent that owns the `EXECUTE` task for data-domain requests.
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every classification decision on a 1–5 rubric.
- **RouterWorkflow** — Workflow per request: classify → guardrail-check → route → execute → publish.
- **RequestEntity** — EventSourcedEntity holding each request's lifecycle.
- **RequestView + RouterEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `ClassificationDecided` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `RESEARCH`, `TRANSLATION`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Request` record with deployer-specific fields (`priority`, `tenantId`, `deadline`).
- `prompts/classifier-agent.md` — tighten the classifier rules (domain-keyword lists, default-to-UNKNOWN thresholds).
- `prompts/content-specialist.md` / `prompts/code-specialist.md` / `prompts/data-specialist.md` — encode output format requirements, length constraints, persona rules.
- `eval-matrix.yaml` — extend the guardrail rubric with deployment-specific rules.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a content-domain request → classified as `CONTENT`, guardrail passes, handed off to `ContentSpecialist`, result published.
2. Simulator drips a code-domain request → classified as `CODE`, guardrail passes, handed off to `CodeSpecialist`, result published.
3. An ambiguous request classifies as `UNKNOWN` and the workflow terminates in `UNROUTABLE` without invoking any specialist.
4. A request that trips the `before-agent-invocation` guardrail (e.g. contains a prompt-injection attempt) is blocked before the specialist is ever called; the request lands in `BLOCKED`.
5. The eval score (1–5) and rationale appear on every classified request within a few seconds of the classification decision.

## License

Apache 2.0.
