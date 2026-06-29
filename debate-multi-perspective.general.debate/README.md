# Akka Sample: Multi-Perspective Debate

A DebateModerator runs up to five alternating rounds between an Advocate and a Critic on a topic you enter, then a Synthesizer writes a balanced conclusion with the key arguments from both sides.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One AI key option (you choose during generation): a mock provider that needs no key, one of `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`, an env file path, a secrets-store URI, or a key typed once into the session. The key value is never written to disk.
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./debate-multi-perspective.general.debate ~/my-projects/debate
cd ~/my-projects/debate
```

(Optional) Edit `SPEC.md`.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- Three AutonomousAgents: `Advocate`, `Critic`, `Synthesizer`.
- One Workflow: `DebateModerator` orchestrating opening, up to five debate rounds, and synthesis.
- One EventSourcedEntity: `DebateEntity` holding the debate lifecycle and rounds.
- One EventSourcedEntity: `InboundRequestQueue` for queued topics.
- One View: `DebatesView` projecting debates for the UI and SSE stream.
- One Consumer: `DebateRequestConsumer` starting a workflow per queued topic.
- One Consumer: `SynthesisEvalConsumer` firing a quality eval on each synthesis.
- One TimedAction: `DebateSimulator` seeding canned topics.
- Two HttpEndpoints: `DebateEndpoint` (API) and `AppEndpoint` (static UI).
- A single self-contained `index.html` UI with five tabs.

## Customise before generating

- System name and model provider — SPEC.md Section 1 and Section 11.
- Topic seed list and round count — SPEC.md Section 11 companion files.
- Governance controls — `eval-matrix.yaml`.

## What gets validated

- A topic submitted via the API drives a debate through alternating rounds to a concluded synthesis (see `reference/user-journeys.md`).
- The synthesis is guardrailed for balance and safety before it reaches the user.
- A quality eval fires on each synthesis and surfaces a score in the UI.
- The background simulator seeds topics and runs debates without UI interaction.

## License

Apache 2.0.
