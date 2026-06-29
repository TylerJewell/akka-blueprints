# Architecture

The system runs a bounded debate between two opposed agents and synthesises the result. The four diagrams below are the source for the Architecture tab; the generated `index.html` renders them with the Lesson 24 state-label CSS overrides so state names and edge labels stay legible on the dark theme.

## Component graph

The flowchart in `PLAN.md` (Component graph) shows the full wiring. A topic enters through `DebateEndpoint` or is dripped by `DebateSimulator` into `InboundRequestQueue`. `DebateRequestConsumer` starts one `DebateModerator` workflow per queued topic. The workflow drives `Advocate`, `Critic`, and `Synthesizer`, writing each transition onto `DebateEntity`. `DebatesView` projects the entity for the UI list and SSE stream; `SynthesisEvalConsumer` reacts to the synthesis event and writes a quality score back onto the entity.

## Interaction sequence

The sequence diagram in `PLAN.md` (Interaction sequence) traces the primary journey: submit → enqueue → start workflow → alternating rounds → guardrailed synthesis → eval. The loop runs up to `maxRounds` (5). The synthesis step is guarded before its result is persisted.

## State machine

The state diagram in `PLAN.md` (State machine) is the `DebateEntity` lifecycle: `PENDING → DEBATING → SYNTHESIZING → CONCLUDED`, with `FAILED` reachable from `DEBATING` or `SYNTHESIZING` (workflow error or a guardrail-blocked synthesis). `DEBATING` self-loops once per recorded round until the rounds are exhausted or the sides converge.

## Entity model

The ER diagram in `PLAN.md` (Entity model) shows `DEBATE` containing many `ROUND` values, projected into `DEBATES_VIEW`, and `INBOUND_REQUEST` starting a `DEBATE`. The view row reuses the `Debate` record; its nullable lifecycle columns are `Optional<T>` so the materializer accepts rows written before synthesis (Lesson 6).
