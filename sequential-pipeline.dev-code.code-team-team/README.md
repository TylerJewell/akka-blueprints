# Akka Sample: Game Builder Team

A simulated software-dev team: a senior engineer agent writes a small Python game, a QA agent runs tests against it, and a chief QA agent gives the final ship-or-rework call. You submit a one-line game brief; the team produces reviewed, tested code.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One AI key option (you choose during generation): mock provider (no key), an existing env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env-file path, a secrets-store URI, or type-once-in-session.
- Host software: none. This blueprint runs out of the box — the code-execution sandbox is modeled inside the same service.

## Generate the system

```sh
cp -r ./sequential-pipeline.dev-code.code-team-team  ~/my-projects/game-builder-team
cd ~/my-projects/game-builder-team
```

(Optional) Edit `SPEC.md` — system name, model provider, the game briefs.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` automatically and prints the listening URL when the service is up.

## What you'll get

- Three pipeline agents: an engineer agent (writes the game), a QA agent (runs tests), a chief QA agent (ship decision).
- A `GameBuildWorkflow` running the three stages in sequence with explicit step timeouts.
- A `BuildEntity` (event-sourced) holding each build's lifecycle, plus an `InboundBriefQueue` entity.
- A `BuildsView` read model streamed to the UI over SSE.
- A `BriefConsumer` and a `BriefSimulator` timer that seed game briefs.
- A guarded in-process code-execution sandbox for the QA test run.
- The 5-tab UI shell: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md` Section 1 — system name and one-line pitch.
- `SPEC.md` Section 11 — model provider and HTTP port.
- `prompts/engineer-agent.md` — the game style and constraints the engineer writes to.
- `src/main/resources/sample-events/game-briefs.jsonl` (created at generation) — the seeded briefs.

## What gets validated

- Submitting a brief produces a build that moves QUEUED → ENGINEERED → QA → SHIPPED with non-empty source code.
- A build whose tests fail lands in REJECTED, never SHIPPED.
- The QA sandbox guardrail blocks code containing forbidden operations before any simulated execution.
- The Eval Matrix tab renders three controls; the Risk Survey tab renders pre-filled answers with deployer placeholders muted.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
