# User journeys — landing-page-builder

Acceptance journeys. Each must pass for the generated system to be correct.

## J1 — Submit a concept and watch it publish

- **Preconditions:** service running on `http://localhost:9364/`; a model
  provider configured (real or mock).
- **Steps:** open the App UI tab; type "a booking page for a mobile bike-repair
  service"; click Submit.
- **Expected state:** the response carries a `pageId`. The page appears in
  `QUEUED`, then advances `ANALYZED → SELECTED → CUSTOMIZED → PUBLISHED` over
  SSE, typically within ~30s.
- **Expected UI:** the row's status pill updates live; the detail view shows a
  brief, a template name, `lintPassed: true`, and a rendered markup preview.
- **Done when:** the page reaches `PUBLISHED` with non-empty `html`.

## J2 — Scrape of an off-allowlist URL is blocked

- **Preconditions:** service running; `TemplateSelector` has a scrape capability.
- **Steps:** drive a concept whose selection step attempts an off-allowlist URL
  (the mock/template path includes one such case).
- **Expected state:** the before-tool-call guardrail blocks the URL; the run
  falls back to a canned source and still produces a `TemplateChoice`.
- **Expected UI:** the page still completes; the chosen `templateUrl` is an
  allowlisted source.
- **Done when:** the page reaches `PUBLISHED` and no off-allowlist URL was
  fetched. A direct `GET /api/scrape?url=<off-list>` returns 403.

## J3 — Unsafe markup is sanitized before storage

- **Preconditions:** service running.
- **Steps:** drive a concept whose customization output contains a `<script>` tag
  (the mock path includes one such case).
- **Expected state:** the before-agent-response guardrail strips the script
  before the `LandingPage` result is persisted.
- **Expected UI:** the stored `html` shown in the detail preview contains no
  `<script>` element.
- **Done when:** the persisted markup has no script, handlers, or external form
  actions.

## J4 — Markup that fails the lint gate is rejected

- **Preconditions:** service running.
- **Steps:** drive a concept whose customization output is missing a `<title>` or
  has an empty body (the mock path includes one such case).
- **Expected state:** `validateStep` lint fails; the page transitions to
  `REJECTED` with a `rejectReason`.
- **Expected UI:** the row shows the `REJECTED` pill and the reason.
- **Done when:** the page is `REJECTED` and never reaches `PUBLISHED`.

## J5 — Simulator seeds a concept with no UI interaction

- **Preconditions:** service running; no user action.
- **Steps:** wait for the `RequestSimulator` 30s tick.
- **Expected state:** a canned concept from `concepts.jsonl` is enqueued; a
  workflow starts and a new page appears.
- **Done when:** a page appears and advances without any user submission.
