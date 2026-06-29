# User journeys — content-pipeline

Acceptance journeys. Each must pass against the generated system. "Done" is the expected end state.

## J1 — Submit a topic and watch it publish

- **Preconditions:** service running on `http://localhost:9942`; one model provider configured (or mock).
- **Steps:**
  1. Open the App UI tab.
  2. Type a topic and click Submit.
  3. Watch the SSE list.
- **Expected state changes:** a new article appears in `RESEARCHING`, then transitions `WRITING` → `CRITIQUING` → `PUBLISHED` (typically within ~60s).
- **Expected UI:** research summary + sources appear at WRITING; draft title + body at CRITIQUING; critique score + notes and a `publishedUrl` at PUBLISHED.
- **Done:** the article is `PUBLISHED` with a non-empty `body` and a non-null `publishedUrl`.

## J2 — Research domain guard blocks an off-list source

- **Preconditions:** `allowed-domains.txt` lists the permitted source domains.
- **Steps:**
  1. Submit a topic whose canned search results include at least one off-list domain.
  2. Inspect the resulting `researchSources`.
- **Expected:** the before-tool-call guard blocks the off-list domain; it does not appear in `researchSources`. Only allowed domains are cited.
- **Done:** the published article's source list contains no off-list domain.

## J3 — Writer style/safety guard blocks a bad draft

- **Preconditions:** mock or model configured to produce a draft that fails the style/safety check (too short, profane, or off-topic).
- **Steps:**
  1. Submit the topic that triggers a failing draft.
  2. Watch the article status.
- **Expected:** the before-agent-response guard blocks the draft; the write step fails over; the article moves to `FAILED` with a `failureReason`. It never reaches `CRITIQUING` or `PUBLISHED`.
- **Done:** the article is `FAILED` with a non-null `failureReason`.

## J4 — Critique score surfaces before publish

- **Preconditions:** a topic that drafts and critiques cleanly.
- **Steps:**
  1. Submit the topic.
  2. Expand the article detail when status reaches CRITIQUING/PUBLISHED.
- **Expected:** a numeric `critiqueScore` (0.0–1.0) and `critiqueNotes` are present and shown in the UI before the `publishedUrl` appears.
- **Done:** `critiqueScore` and `critiqueNotes` are non-null on the published article.

## J5 — Background load from the simulator

- **Preconditions:** service running; no UI interaction.
- **Steps:** wait for `TopicSimulator` to drip a topic from `sample-events/topics.jsonl`.
- **Expected:** a fresh article appears and runs the full pipeline without any user action.
- **Done:** at least one simulator-seeded article reaches `PUBLISHED`.
