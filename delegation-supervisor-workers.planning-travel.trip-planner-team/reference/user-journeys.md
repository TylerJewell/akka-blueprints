# User journeys

Acceptance tests as numbered journeys. Passing all of them means the blueprint generated correctly.

## J1 — Plan a trip end to end

- **Preconditions:** service running on `http://localhost:9385/`; a model provider key set, or the mock provider chosen.
- **Steps:** in the App UI tab, enter destination `Lisbon`, preferences `food, walking, slow pace`, days `3`; submit.
- **Expected state:** the trip appears in `GATHERING_INSIGHTS`, then advances `PLANNING_ITINERARY` → `PLANNING_LOGISTICS` → `COMPLETED` within ~60s.
- **Expected UI:** the four sections (Insights, Itinerary, Logistics, Final plan) fill in order; the final card shows a non-empty `finalPlan`.
- **Done when:** `GET /api/trips/{id}` returns `status = COMPLETED` with all four lifecycle fields present.

## J2 — Scrape guard blocks a non-allowlisted URL

- **Preconditions:** a trip in progress whose local-expert attempts to scrape a URL outside `travel.allowed-hosts`.
- **Steps:** observe the gather step (or call `GET /api/sim/scrape?url=https://evil.example/x` directly).
- **Expected:** the scrape endpoint returns `{ "blocked": true, ... }`; the agent proceeds with search-only insights.
- **Done when:** the trip still reaches `PLANNING_ITINERARY` with non-empty `localInsights` and the blocked URL is absent from `sources`.

## J3 — Grounding guard blocks an ungrounded itinerary

- **Preconditions:** an itinerary draft names a place absent from the gathered insights (forceable via the mock provider).
- **Steps:** run a trip whose first itinerary draft is ungrounded.
- **Expected:** the before-agent-response guard rejects the draft; the step retries (bounded by `maxRetries`).
- **Done when:** the trip reaches `COMPLETED` with an itinerary whose every named place appears in `localInsights`, or `FAILED` if retries are exhausted.

## J4 — Background load from the simulator

- **Preconditions:** service running; no UI interaction.
- **Steps:** wait ~30s for `TripRequestSimulator` to drip a request from `sample-events/trip-requests.jsonl`.
- **Expected:** `TripRequestConsumer` starts a workflow; a new trip appears and runs to `COMPLETED`.
- **Done when:** `GET /api/trips` lists at least one simulator-seeded trip in `COMPLETED`.
