# Akka Sample: CAMEL Role-Play Collaboration

A Coordinator assigns a task to an AI Assistant and an AI User, then moderates up to twelve rounds of structured dialogue between them until the task is solved, the parties reach an impasse, or the round cap is hit.

This folder is a **blueprint** — a set of natural-language and YAML inputs. You run `/akka:specify` against it and Claude generates a working Akka project. The blueprint itself contains no Java, no `pom.xml`, and no built UI.

## Prerequisites

- **Claude Code** with the **Akka plugin** installed. Install docs: <https://doc.akka.io/>.
- **One model-provider API key**, sourced however you prefer. When you run `/akka:specify`, Claude detects which of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` is set and wires it. If none is set, Claude asks how you want to source the key and offers five options:
  - a mock provider that returns schema-valid dialogue turns with no key,
  - name an existing environment variable,
  - point at an env file you already maintain,
  - a secrets-store reference (`1password://…`, `aws-secretsmanager://…`, `vault://…`),
  - type the key once into the session.
  The key value is never written to any file Claude creates — only the reference is recorded.
- **Host software:** none. This blueprint runs out of the box; the Assistant, the User, and the Coordinator are all modeled inside the one service.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — the system name, the model provider, or the role-play defaults (task description, domain, convergence criteria, round cap).
3. In Claude Code, from inside the folder, run:

   ```
   /akka:specify @SPEC.md
   ```

That is the only command you type. `SPEC.md` Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` on its own and to print the listening URL when the service is up.

## What you'll get

- **AssistantAgent** and **UserAgent** — two agents that each produce a typed dialogue turn per round, staying within their assigned roles and instructions.
- **CoordinatorAgent** — the moderator that reads both latest turns each round and decides continue, solved, or impasse, summarising the solution on convergence.
- **CollaborationWorkflow** — the durable turn-taking loop that alternates User → Assistant → Coordinator for up to twelve rounds.
- **CollaborationEntity** — the event-sourced record of every turn and the concluded outcome.
- **SolutionEvaluator** — scores each concluded collaboration (solution quality, rounds used) when it ends.
- **CollaborationsView**, a request queue, a request simulator, a stalled-collaboration watch, an operator halt switch, and the two HTTP endpoints that serve the API and the embedded five-tab UI.

## Customise before generating

The parts of `SPEC.md` most worth editing before you run `/akka:specify`:

- **System name** — Section 1, plus the UI title in Section 7.
- **Model provider** — Section 11's identity block; or just set the matching env var and let Claude detect it.
- **Role-play defaults** — Section 5 and `reference/data-model.md`: the convergence criteria, the twelve-round cap, and the canned tasks in the request simulator.
- **Agent behavior** — the three files under `prompts/` carry the Assistant, User, and Coordinator system prompts.

## What gets validated

The generated system is correct when the journeys in `reference/user-journeys.md` pass:

1. Submitting a collaboration task starts a workflow and the User's first dialogue turn appears within seconds.
2. A well-posed task with a clear solution domain reaches `CONCLUDED` with `outcome = SOLVED` and a non-empty `solutionSummary`.
3. A task with no achievable convergence ends in `IMPASSE` at or before round twelve.
4. Every concluded collaboration receives an outcome score from the evaluator.

## License

Apache 2.0.
