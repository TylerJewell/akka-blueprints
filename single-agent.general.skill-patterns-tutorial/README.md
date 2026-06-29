# Akka Sample: Skill Patterns Tutorial

A single agent demonstrates four distinct skill-wiring patterns in one runnable service: an **inline skill** defined entirely inside the agent definition, a **file-based skill** whose prompt is loaded from a classpath resource, an **external skill** that calls a lightweight HTTP tool before composing its response, and a **meta skill-creator** that reads a user-supplied description and emits a new skill definition the agent can use on the next request.

Each pattern is triggered through a dedicated endpoint. The agent produces a `SkillResult` carrying the pattern name, the skill's output text, and a short explanation of how the skill was wired so the reader can see the mechanism, not just the output.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. All four skill patterns run in-process; the external-skill pattern targets a small in-process mock HTTP stub that starts with the service.

## Generate the system

```sh
cp -r ./single-agent.general.skill-patterns-tutorial  ~/my-projects/skill-patterns-tutorial
cd ~/my-projects/skill-patterns-tutorial
```

(Optional) Edit `SPEC.md §3` to change the seeded inline-skill prompt, swap the file-based skill's resource path, or adjust the external tool's stub behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SkillDemoAgent** — an AutonomousAgent that hosts all four skill patterns and dispatches to the right one based on the incoming task.
- **SkillRunEntity** — an EventSourcedEntity that records the lifecycle of every skill invocation: requested → running → completed or failed.
- **SkillRunWorkflow** — a Workflow that coordinates the run: routes to the agent, waits for the result, and records it on the entity.
- **SkillRunView** — a View projecting entity events into rows for the UI list.
- **SkillRunEndpoint + AppEndpoint** — REST/SSE API and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seed skills (inline prompt text, resource file path, external tool URL, or meta-creator input schema).
- `SPEC.md §5` — extend `SkillResult` with domain-specific fields (e.g., a `confidence` float or a `sourcePattern` tag).
- `prompts/skill-demo-agent.md` — tighten the dispatch rules or add a fifth pattern to the routing table.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user invokes each of the four skill patterns via the App UI and receives a `SkillResult` with the correct `patternName` and a non-empty `output`.
2. The meta skill-creator pattern accepts a user-supplied description and returns a new skill definition; a subsequent request using that definition produces output.
3. The external skill pattern calls the in-process stub; the call log shows the outbound HTTP request and the returned value embedded in the result.
4. All four patterns' results appear in the live list as they complete, newest-first.

## License

Apache 2.0.
