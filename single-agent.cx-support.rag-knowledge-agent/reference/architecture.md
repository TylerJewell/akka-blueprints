# Architecture — rag-knowledge-agent

One request/response agent grounded on a retrieved passage set, with a per-answer faithfulness eval running off to the side. Retrieval runs in-process over a bundled document set, so the system runs out of the box.

The four diagrams below are the mermaid sources the generated UI renders on the Architecture tab. Render them with the Lesson 24 state-label CSS overrides and theme variables so state names and edge labels stay legible on the dark background.

## Component graph

The graph in `PLAN.md` ("Component graph") shows the runtime wiring. `KnowledgeEndpoint` is the orchestration point: it records the query session, searches `DocIndexEntity`, calls `KnowledgeAgent`, and records the answer or refusal. `DocLoader` populates the index on a timer. `FaithfulnessEvalConsumer` reacts to the `QueryAnswered` event and runs `FaithfulnessAgent`, closing the loop by recording the score back on the session.

## Interaction sequence

The sequence in `PLAN.md` ("Interaction sequence") is the primary journey: ask → retrieve → answer (or refuse) → evaluate. The grounding guardrail fires inside `KnowledgeAgent` before its answer leaves the agent; an unsupported answer becomes a `QueryRefused` branch. The faithfulness eval is downstream and non-blocking, so the user sees the answer immediately and the score arrives a moment later over SSE.

## State machine

The lifecycle in `PLAN.md` ("State machine") tracks one query session: `RECEIVED → RETRIEVED → ANSWERED → EVALUATED`, with `RETRIEVED → REFUSED` as the terminal off-source branch. `REFUSED` and `EVALUATED` are terminal.

## Entity model

The ER diagram in `PLAN.md` ("Entity model") shows the two state holders: `QuerySessionEntity` (event-sourced, one stream per question, projected into `QuerySessionView`) and `DocIndexEntity` (key-value, one well-known instance holding all chunks). Citations are embedded in the session; chunks are embedded in the index.
