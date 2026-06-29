# Akka Sample: Conversational Interviews via AI Form

An AI agent conducts structured, adaptive interviews by presenting questions one at a time through a conversational form. Each response is considered before the next question is chosen — the agent branches based on what the candidate says, not on a fixed script. Submitted responses are screened for protected-category attributes before the agent ever processes them, and every question the agent intends to ask is checked for bias or legal risk before it is shown to the candidate.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a special-category sanitizer that runs before the agent processes any submitted answer, and a `before-agent-response` guardrail that validates every outgoing question for bias and legal compliance before it reaches the candidate.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The interview session runs in-process; the agent's question selection and seeded role definitions live in-process.

## Generate the system

```sh
cp -r ./single-agent.hr-recruiting.conversational-interview  ~/my-projects/conversational-interview
cd ~/my-projects/conversational-interview
```

(Optional) Edit `SPEC.md` to point at a different role definition (e.g., switch from the seeded software-engineer interview guide to a product-manager or sales-representative guide).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InterviewConductorAgent** — an AutonomousAgent that receives a role definition and the conversation history and decides the next question to ask the candidate.
- **InterviewWorkflow** — orchestrates sanitize-wait → conduct-turn → complete per submitted answer.
- **InterviewSessionEntity** — an EventSourcedEntity holding the per-session lifecycle from `OPEN` through `COMPLETED`.
- **AnswerSanitizer** — a Consumer that subscribes to `AnswerSubmitted` events, screens for protected-category attributes, and emits `AnswerScreened` back to the entity.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded role definitions for your own (the JSONL file under `src/main/resources/sample-events/role-definitions.jsonl` after generation).
- `SPEC.md §5` — extend `InterviewTurn` with role-specific fields (e.g., `competencyTag`, `scoringRubricId`, `followUpDepth`).
- `prompts/interview-conductor.md` — narrow the agent's scope (a technical-screen deployer might restrict to engineering-specific competency frameworks; a behavioural-screen deployer to STAR-method prompts only).
- `eval-matrix.yaml` — replace the in-process special-category screen with an external identity-verification or HR-data processor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A coordinator starts a session for a role → the agent generates an opening question → the candidate answers → the agent generates the next adaptive question → the session reaches `COMPLETED` with a transcript.
2. A candidate's answer contains a reference to a protected attribute → the `AnswerSanitizer` redacts it → the agent's next question never references the protected content.
3. The agent drafts a question containing language that fails the bias check → the `before-agent-response` guardrail rejects it → the agent retries → the revised question is legal and bias-free.
4. A completed session's transcript shows every agent-generated question, every candidate answer, and the sanitizer's screening result per turn — the raw answer with the protected-category reference is preserved on the entity for audit but is never the input seen by the model.

## License

Apache 2.0.
