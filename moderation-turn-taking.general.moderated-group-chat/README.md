# Akka Sample: Moderated Group Chat

Multiple AI assistants converse in a shared chat session, each taking a turn in sequence under a chat orchestrator. The orchestrator checks every reply before it is added to the shared transcript and ends the session when a termination condition is met.

This folder is a **blueprint** — a set of natural-language and YAML inputs. You run `/akka:specify` against it and Claude generates a working Akka project. The blueprint itself contains no Java, no `pom.xml`, and no built UI.

## Prerequisites

- **Claude Code** with the **Akka plugin** installed. Install docs: <https://doc.akka.io/>.
- **One model-provider API key**, sourced however you prefer. When you run `/akka:specify`, Claude detects which of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` is set and wires it. If none is set, Claude asks how you want to source the key and offers five options:
  - a mock provider that returns schema-valid chat messages with no key,
  - name an existing environment variable,
  - point at an env file you already maintain,
  - a secrets-store reference (`1password://…`, `aws-secretsmanager://…`, `vault://…`),
  - type the key once into the session.
  The key value is never written to any file Claude creates — only the reference is recorded.
- **Host software:** none. This blueprint runs out of the box; the assistant personas, the chat session, and the termination logic are all modeled inside the one service.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — the system name, the model provider, the assistant personas, or the chat defaults (max turns, turn timeout, termination keywords).
3. In Claude Code, from inside the folder, run:

   ```
   /akka:specify @SPEC.md
   ```

That is the only command you type. `SPEC.md` Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` on its own and to print the listening URL when the service is up.

## What you'll get

- **ResearcherAssistant** and **CriticAssistant** — two assistant agents each producing a typed `ChatTurn` per round, held to their own persona constraints.
- **OrchestratorAgent** — the moderator that reads the full transcript each round and decides continue, conclude, or escalate, proposing a summary on conclusion.
- **ChatSessionWorkflow** — the durable turn-taking loop that alternates Researcher → Critic → Orchestrator for up to twenty turns.
- **ChatSessionEntity** — the event-sourced record of every turn and the concluded session state.
- **SessionQualityEvaluator** — scores each concluded session (topic coherence, turn depth, termination reason) when it ends.
- **SessionsView**, a request queue, a request simulator, a stalled-session watch, an operator halt switch, and the two HTTP endpoints that serve the API and the embedded five-tab UI.

## Customise before generating

The parts of `SPEC.md` most worth editing before you run `/akka:specify`:

- **System name** — Section 1, plus the UI title in Section 7.
- **Model provider** — Section 11's identity block; or just set the matching env var and let Claude detect it.
- **Chat defaults** — Section 5 and `reference/data-model.md`: the twenty-turn cap, turn timeout, and the canned topics in the request simulator.
- **Assistant personas** — the three files under `prompts/` carry the Researcher, Critic, and Orchestrator system prompts.

## What gets validated

The generated system is correct when the journeys in `reference/user-journeys.md` pass:

1. Starting a chat session creates a workflow and the Researcher's first turn appears within seconds.
2. A session on a well-scoped topic reaches `CONCLUDED` with `terminationReason = CONSENSUS` and a non-empty summary.
3. A session on a deliberately irresolvable topic concludes with `terminationReason = MAX_TURNS_REACHED` at or before turn twenty.
4. Every concluded session receives a quality score from the evaluator.

## License

Apache 2.0.
