# Akka Sample: Pokemon Team Advisor

A single team-advisor agent receives a trainer's current roster and a target battle format, then returns a structured team recommendation: a scored team composition with per-slot rationale and coverage gaps. The roster rides into the agent as a task attachment, not as inline prompt text.

Demonstrates the **single-agent** coordination pattern in a low-stakes educational domain. One `TeamAdvisorAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its input, validate its output structure, and score the recommendation quality.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. All Pokemon data lives in-process; no external game data APIs are called.

## Generate the system

```sh
cp -r ./single-agent.general.pokemon-advisor  ~/my-projects/pokemon-advisor
cd ~/my-projects/pokemon-advisor
```

(Optional) Edit `SPEC.md` to change the seeded roster examples (e.g., swap the competitive VGC team for a casual in-game team) or add a different battle format.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TeamAdvisorAgent** — an AutonomousAgent that accepts the trainer's roster and battle format as a task attachment and returns a typed `TeamRecommendation`.
- **AdvisoryWorkflow** — orchestrates validate-wait → advise → score per submitted roster.
- **RosterEntity** — an EventSourcedEntity holding the per-advisory lifecycle.
- **RosterValidator** — a Consumer that subscribes to `RosterSubmitted` events, checks basic legality (no duplicate species, within slot count limits), and emits `RosterValidated` back to the entity.
- **AdvisoryView + AdvisoryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded roster examples for your own battle format (the JSONL file under `src/main/resources/sample-events/rosters.jsonl` after generation).
- `SPEC.md §5` — extend `TeamRecommendation` with format-specific fields (e.g., `restrictedLegendaryAllowed`, `seriesRuleset`, `pointsBudget`).
- `prompts/team-advisor.md` — narrow the agent's role (a competitive player would constrain it to VGC Series rules; a casual player to story-mode coverage).
- `eval-matrix.yaml` — wire a real type-coverage library by naming it under the validator mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A trainer submits a roster + format → it is validated → advised → the recommendation appears in the UI.
2. The agent returns a malformed recommendation on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed recommendation lands.
3. Every recorded recommendation has a coverage-completeness score visible on the same UI card.
4. A roster containing a banned species for the selected format is flagged in validation and never reaches the agent.

## License

Apache 2.0.
