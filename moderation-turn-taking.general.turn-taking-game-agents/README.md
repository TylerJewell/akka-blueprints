# Akka Sample: AgentChess

An AI player and an opponent — a human at the keyboard or a random mover — take alternating turns at chess, with every proposed move checked for legality against the current board before it is applied.

This folder is a **blueprint** — a set of natural-language and YAML inputs. You run `/akka:specify` against it and Claude generates a working Akka project. The blueprint itself contains no Java, no `pom.xml`, and no built UI.

## Prerequisites

- **Claude Code** with the **Akka plugin** installed. Install docs: <https://doc.akka.io/>.
- **One model-provider API key**, sourced however you prefer. When you run `/akka:specify`, Claude detects which of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` is set and wires it. If none is set, Claude asks how you want to source the key and offers five options:
  - a mock provider that returns legal moves with no key,
  - name an existing environment variable,
  - point at an env file you already maintain,
  - a secrets-store reference (`1password://…`, `aws-secretsmanager://…`, `vault://…`),
  - type the key once into the session.
  The key value is never written to any file Claude creates — only the reference is recorded.
- **Host software:** none. This blueprint runs out of the box; the board, the chess rules, the move validator, the opponent, and the operator halt switch are all modeled inside the one service.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — the system name, the model provider, the AI side, or the opponent type (human or random).
3. In Claude Code, from inside the folder, run:

   ```
   /akka:specify @SPEC.md
   ```

That is the only command you type. `SPEC.md` Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` on its own and to print the listening URL when the service is up.

## What you'll get

- **ChessPlayerAgent** — an agent that proposes the AI side's move given the current board position, returning a typed move.
- **MoveValidator** — the in-process chess-rules engine: it checks a proposed move against the current board, returns the resulting position, and reports checkmate, stalemate, or draw.
- **GameWorkflow** — the durable turn-taking referee that alternates the two sides, validates each move before applying it, and ends the game on a terminal position.
- **GameEntity** — the event-sourced record of every applied move and the game result.
- **GamesView**, a request queue, a request simulator, a stalled-game watch, an operator halt switch, and the two HTTP endpoints that serve the API and the embedded five-tab UI.

## Customise before generating

The parts of `SPEC.md` most worth editing before you run `/akka:specify`:

- **System name** — Section 1, plus the UI title in Section 7.
- **Model provider** — Section 11's identity block; or just set the matching env var and let Claude detect it.
- **Game defaults** — Section 5 and `reference/data-model.md`: the AI side (White or Black), the opponent type (human or random), and the canned games in the request simulator.
- **Agent behavior** — `prompts/chess-player-agent.md` carries the player's system prompt.

## What gets validated

The generated system is correct when the journeys in `reference/user-journeys.md` pass:

1. Starting a game against the random opponent produces a legal first move within seconds.
2. Every applied move is legal for the side to move; no illegal move ever reaches the board.
3. A game reaches a terminal result — checkmate, stalemate, or draw — and records the winner and the reason.
4. A human can submit a move and an illegal one is rejected with a reason rather than applied.

## License

Apache 2.0.
