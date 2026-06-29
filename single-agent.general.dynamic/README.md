# Akka Sample: Dynamic Route Agent

One generic agent class, configured per request, runs both a SUMMARIZE route and a TRANSLATE route from the same code.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) -> "Spec-Driven Development with Claude Code".
- An AI key option — one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` set in your shell. If none is set, `/akka:specify` offers a mock-model path that needs no key.
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./single-agent.general.dynamic  ~/my-projects/dynamic-route-agent
cd ~/my-projects/dynamic-route-agent
```

(Optional) Edit `SPEC.md` to change the system name, the model provider, or the route definitions.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. Section 12 of `SPEC.md` instructs Claude to continue automatically through `/akka:plan` -> `/akka:tasks` -> `/akka:implement` -> `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `DynamicAgent` — one `Agent` class configured per request with route-specific instructions.
- `RequestEntity` — an `EventSourcedEntity` holding each request's lifecycle.
- `RequestsView` — a `View` read model the UI lists and streams over SSE.
- `RouteEndpoint` — the `/api` surface: submit, list, single, SSE, metadata.
- `AppEndpoint` — serves the single-file UI.
- `RequestSimulator` — a `TimedAction` that drips sample requests for background activity.
- Two guardrails: one before the agent runs (blocks unsafe per-request configurations), one before the response leaves (blocks unsafe output).

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider, HTTP port, and the per-route instruction text.
- `prompts/dynamic-agent.md` — the base instruction the agent runs with under each route.

## What gets validated

- Submitting a SUMMARIZE request returns a shorter output and drives the request to `COMPLETED`.
- Submitting a TRANSLATE request with a target language returns translated text and drives it to `COMPLETED`.
- A request with an unsafe configuration is stopped before the agent runs and lands in `BLOCKED`.
- The simulator seeds requests on its own and they appear over SSE without any user action.

Full journeys live in `reference/user-journeys.md`.

## License

Apache 2.0.
