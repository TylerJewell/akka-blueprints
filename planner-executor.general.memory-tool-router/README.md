# Akka Sample: AI Chatbot with Long-Term Memory and Dynamic Tool Routing

A chatbot that maintains a per-conversation long-term memory store and dynamically selects the right tool for each user turn. Demonstrates the **planner-executor** coordination pattern with a RouterAgent that classifies each turn, a MemoryAgent that reads and writes structured memories, and a set of registered tool-executor agents (Calculator, KnowledgeBase, WebLookup, CodeRunner). Governance controls gate every tool call for policy compliance and scrub PII before memories are persisted.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The four tool-executor surfaces — calculator, knowledge base, web lookup, code runner — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.general.memory-tool-router  ~/my-projects/memory-tool-router
cd ~/my-projects/memory-tool-router
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RouterAgent** — AutonomousAgent that classifies each user message into a `RoutingDecision` (which tool, or no tool), checks the tool allowlist via a `before-tool-call` guardrail, and returns a `TurnPlan`.
- **MemoryAgent** — AutonomousAgent that reads the session's memory store before each reply, writes new memories after each reply, and applies a PII sanitizer before any memory is persisted.
- **CalculatorAgent** — AutonomousAgent that evaluates mathematical expressions from seeded fixtures.
- **KnowledgeBaseAgent** — AutonomousAgent that answers factual queries from seeded knowledge fixtures.
- **WebLookupAgent** — AutonomousAgent that returns lookup answers from seeded web snippets.
- **CodeRunnerAgent** — AutonomousAgent that returns simulated code execution output for an allow-listed expression set.
- **ConversationWorkflow** — Workflow that drives the receive → recall → route-and-guard → execute → sanitize-memory → store → reply loop, with graceful halt and stuck-conversation branches.
- **ConversationEntity** — EventSourcedEntity holding the conversation lifecycle, message history, and memory references.
- **MemoryEntity** — EventSourcedEntity holding the per-session structured memory store.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag. Single instance keyed by literal `"global"`.
- **MessageQueue** — EventSourcedEntity audit log of submitted messages.
- **ConversationView** — projection used by the UI.
- **MessageQueueConsumer** — Consumer that starts a ConversationWorkflow turn per submitted message.
- **SessionSimulator** — TimedAction that drips a sample conversation turn every 90 s so the App UI is not empty when first loaded.
- **StuckConversationMonitor** — TimedAction that marks any conversation stuck in `PROCESSING` past 4 minutes as `STUCK`.
- **ConversationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample conversation turns the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Memory` record fields (e.g., add a `confidenceScore`).
- `prompts/router.md` — narrow the router to a specific tool policy.
- `eval-matrix.yaml` — add a `sanitizer-output` control if you want reply-level PII scrubbing in addition to memory-level.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a message → router classifies, tool executes, reply appears within ~30 s. Memory store gains a new fact.
2. Submit a message whose routing would invoke a blocked tool — the guardrail rejects the dispatch; the RouterAgent picks an alternative or the turn ends with a polite refusal.
3. Submit a message containing a PII-shaped string — the memory written after the turn must not contain the raw PII; the sanitizer tag must appear instead.
4. Click **Halt new dispatches** during an active turn; the in-flight tool call finishes; no further turns dispatch until Resume.

## License

Apache 2.0.
