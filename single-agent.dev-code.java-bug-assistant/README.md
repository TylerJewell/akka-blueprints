# Akka Sample: Software Bug Assistant (Java)

A single bug-resolution agent accepts a bug report, searches a connected ticketing system and web sources for related issues and known fixes, and returns a structured resolution recommendation — RESOLVED / NEEDS_INVESTIGATION / ESCALATE — with a ranked list of candidate fixes. The bug report is passed to the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern in the dev-code domain. One `BugResolutionAgent` (AutonomousAgent) carries the entire diagnosis; surrounding components prepare its input, guard its output, and score every recommendation before it surfaces.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — a seeded in-process ticket store provides the ticketing context and the agent's tool calls target the same in-process store, so no external ticket system is required.

## Generate the system

```sh
cp -r ./single-agent.dev-code.java-bug-assistant  ~/my-projects/java-bug-assistant
cd ~/my-projects/java-bug-assistant
```

(Optional) Edit `SPEC.md` to point at a different bug taxonomy (e.g., switch from generic software defects to security CVEs or infrastructure incidents).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BugResolutionAgent** — an AutonomousAgent that accepts a bug report as a task attachment and a set of search results as task instructions, then returns a typed `ResolutionRecommendation`.
- **ResolutionWorkflow** — orchestrates normalize-wait → search → resolve → eval per submitted bug.
- **BugEntity** — an EventSourcedEntity holding the per-bug lifecycle from submission through recommendation.
- **BugNormalizer** — a Consumer that subscribes to `BugSubmitted` events, normalizes the report (strips credentials, stack-trace tokens, and internal hostnames), and emits `BugNormalized` back to the entity.
- **TicketSearcher** — a Consumer that subscribes to `BugNormalized` events, queries the in-process ticket store for related open issues, and emits `SearchResultsAttached`.
- **BugView + BugEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded bug taxonomy for your own (the JSONL file under `src/main/resources/sample-events/bug-taxonomy.jsonl` after generation).
- `SPEC.md §5` — extend `ResolutionRecommendation` with environment-specific fields (e.g., `affectedService`, `deploymentRegion`, `severitySlo`).
- `prompts/bug-resolution-agent.md` — narrow the agent's role (a platform team would constrain it to infrastructure incidents; a security team to CVE triage).
- `eval-matrix.yaml` — replace the in-process ticket store with a live one by naming the integration under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a bug report → it is normalized → related tickets are searched → a recommendation appears in the UI with ranked candidate fixes.
2. The agent returns a malformed recommendation on first try → the `before-tool-call` guardrail blocks the ticket-write → the agent revises its plan → a well-formed recommendation lands.
3. Every recorded recommendation has an on-decision eval score visible on the same UI card.
4. Internal hostnames and stack-trace tokens submitted in the bug report never appear in the LLM call log; only the normalized form does.

## License

Apache 2.0.
