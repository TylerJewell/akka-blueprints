# Akka Sample: Game Builder Team

Type a game idea; a director delegates design to a game designer and code to a code writer, runs the generated code through a sandbox-guarded test gate, and returns a playable game spec plus its code.

## Prerequisites

- Claude Code with the Akka plugin installed — see https://doc.akka.io/.
- An AI model key, or no key at all. When you run the generator, it detects which of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` is set and uses that provider. If none is set, it offers five ways to source a key — including a mock provider that needs no key (see `SPEC.md` Section 11).
- Host software: None. This blueprint runs out of the box. The code-execution and test surfaces are modeled inside the same Akka service, so there is no Docker, no browser, and no external sandbox to install.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — system name, model provider, or the kinds of games the team builds.
3. In Claude Code, run:

   ```
   /akka:specify @SPEC.md
   ```

That is the only command you type. `SPEC.md` Section 12 chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` automatically and prints the listening URL when the service is up.

## What you'll get

- Three agents: `GameDirector` (supervisor), `GameDesigner` (worker), `CodeWriter` (worker).
- A `GameBuildWorkflow` orchestrating design → code → test → assemble.
- A `GameProjectEntity` (event-sourced) holding each build's lifecycle, plus a `BuildRequestQueue` entity for replay.
- A `GameView` read model, a `BuildRequestConsumer`, and a `RequestSimulator` that drips sample prompts.
- Two HTTP endpoints — the game API with SSE, and the app endpoint serving the embedded UI and metadata.
- A 5-tab UI: Overview, Architecture, Risk Survey, Eval Matrix, App UI.

## Customise before generating

- **System name / title** — `SPEC.md` Sections 1 and 7.
- **Model provider** — `SPEC.md` Section 11 ("Generation workflow"); the default follows whichever key env var is set.
- **Game domain specifics** — the prompt files under `prompts/` set what each agent designs and writes.

## What gets validated

The generated system must pass the journeys in `reference/user-journeys.md`:

- Submit a game idea and watch it reach `DELIVERED` with a spec and runnable code.
- A build whose code trips the sandbox guardrail moves to `BLOCKED`.
- A build whose code fails the test gate retries, then moves to `TEST_FAILED`.
- The simulator seeds a build with no user interaction.

## License

Apache 2.0.
