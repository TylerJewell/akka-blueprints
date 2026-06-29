# User journeys — surprise-trip-planner

Acceptance journeys. Passing all of them means the blueprint generated correctly.

## J1 — Submit preferences and get a surprise itinerary

- **Preconditions:** service running; a model provider configured (real or mock).
- **Steps:** in App UI, enter vibe "coastal, slow", budget "mid", a 4-day window, party size 2, no-go "long-haul flights". Click **Plan my surprise**.
- **Expected state:** a `Trip` appears in `REQUESTED`, moves to `RESEARCHED`, then `READY` within ~30s. On `READY` the destination, a `Logistics` block, and a non-empty `activities` list are present.
- **Expected UI:** the trip card masks the destination until `READY`, then reveals all three worker outputs.
- **Done when:** status is `READY` with all worker outputs populated.

## J2 — A worker tool call out of scope is blocked (G1)

- **Preconditions:** service running.
- **Steps:** submit a preference set whose canned path drives a worker to attempt a tool call outside its granted scope (e.g. the activities worker reaching a booking-commit tool).
- **Expected state:** the before-tool-call guardrail blocks the call; a `ToolCallBlocked` event is recorded; `blockedToolNote` is set; the worker falls back and the trip still reaches `READY`.
- **Expected UI:** the trip card shows a small footnote that a tool call was blocked.
- **Done when:** the trip completes and `blockedToolNote` is non-null.

## J3 — An ungrounded travel fact is corrected (G2)

- **Preconditions:** service running.
- **Steps:** submit a preference set whose assembled itinerary would state a visa or price claim unsupported by the reference data.
- **Expected state:** the before-agent-response guardrail strips or flags the claim; `groundingNotes` records the adjustment before the itinerary is persisted.
- **Expected UI:** the `READY` card shows the grounding footnote.
- **Done when:** `groundingNotes` is non-null and the persisted itinerary contains no ungrounded claim.

## J4 — A stalled trip escalates

- **Preconditions:** service running; a trip left in `REQUESTED` or `RESEARCHED` (e.g. a worker step is slow).
- **Steps:** wait past the 3-minute stall threshold without the trip completing.
- **Expected state:** `StalledTripMonitor` calls `markEscalated`; the trip moves to `ESCALATED` with `escalatedAt` set.
- **Expected UI:** the trip card shows `ESCALATED` without any user interaction.
- **Done when:** status is `ESCALATED`.

## J5 — Background load from the simulator

- **Preconditions:** service running; no UI interaction.
- **Steps:** wait. `RequestSimulator` drips a preference set from `trip-requests.jsonl` every 30s.
- **Expected state:** each drip enqueues a request and starts a fresh `TripWorkflow`.
- **Done when:** trips appear and progress without any client request.
