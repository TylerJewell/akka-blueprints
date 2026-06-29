# Akka Sample: Landing Page Generator

> Turn a one-line product concept into a complete landing page — copy, page structure, and call-to-action sections — through a sequential agent pipeline with a brand-safety review gate before the page is marked ready.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI key, sourced one of five ways at generation time (mock provider needs no key): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` already in your shell; a named env var; an env file path; a secrets-store URI; or typed once into the Claude session. The key value is never written to disk.
- Host software: none. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.landing-page-generator  ~/my-projects/landing-page-generator
cd ~/my-projects/landing-page-generator
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the brand-safety review rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding.

## What you'll get

- Three `AutonomousAgent`s: a copy agent, a structure agent, and a call-to-action agent, run in sequence.
- One `Workflow` orchestrating the copy → structure → cta → review pipeline.
- One `EventSourcedEntity` holding each landing page's lifecycle, plus an inbound concept queue entity.
- One `View` projecting pages into a read model streamed to the UI over SSE.
- A `Consumer` that starts a pipeline per inbound concept, and a `TimedAction` that drips sample concepts in the background.
- Two `HttpEndpoint`s: the JSON API and the static UI host.
- A single self-contained UI with five tabs (Overview / Architecture / Risk Survey / Eval Matrix / App UI).

## Customise before generating

- **System name / model provider** — SPEC.md Section 1 and Section 11.
- **Brand-safety review rules** — `prompts/copy-agent.md` and the guardrail rationale in `eval-matrix.yaml` (control G1).
- **Sample concepts** — the concept list the background simulator drips (SPEC.md Section 11, companion files).

## What gets validated

The generated system must pass the journeys in `reference/user-journeys.md`: submit a concept and watch it run through copy/structure/cta to a `READY` page; a page that fails the brand-safety gate lands in `FLAGGED`; the background simulator seeds concepts without UI interaction; and all five UI tabs render.

## License

Apache 2.0.
