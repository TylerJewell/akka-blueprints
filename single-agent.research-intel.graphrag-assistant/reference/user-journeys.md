# User journeys — graphrag-assistant

Acceptance journeys. Passing all of them means the blueprint generated correctly.

## J1 — Local-scope question resolves with citations

- **Preconditions:** service running on `http://localhost:9986/`; `IndexBuilder`
  has built `CorpusIndex` from the bundled docs.
- **Steps:** on the App UI tab, ask a specific named-entity question covered by
  one document (e.g. "What version introduced feature X?").
- **Expected:** a `QueryEntity` is created `RECEIVED`, then transitions to
  `ANSWERED` with `scope = "local"`, non-empty `answer`, `grounded = true`, and
  1–3 citation doc-ids. The row updates live over SSE.

## J2 — Global-scope question uses community summaries

- **Preconditions:** as J1.
- **Steps:** ask a broad thematic question ("What are the main themes across the
  corpus?").
- **Expected:** the query reaches `ANSWERED` with `scope = "global"`, an
  aggregate answer, `grounded = true`, and citations spanning more than one doc.

## J3 — Ungrounded question is blocked

- **Preconditions:** as J1.
- **Steps:** ask a question the corpus does not cover ("What is the capital of a
  country not in any document?").
- **Expected:** the grounding guardrail (G1) fires; the query terminates in
  `BLOCKED` with a `blockedReason` and no fabricated answer. The App UI shows the
  block reason.

## J4 — PII never reaches the answer

- **Preconditions:** one bundled doc contains an obvious PII string (an email or
  phone number).
- **Steps:** ask a question whose best chunk is that doc.
- **Expected:** the sanitizer (S1) redacts the PII before the prompt; the returned
  `answer` contains no raw email or phone number — redactions read as
  `[redacted]`. The query still resolves `ANSWERED`.

## J5 — Index builds once at startup

- **Preconditions:** fresh service start.
- **Steps:** call `GET /api/queries` immediately, then ask a question.
- **Expected:** `IndexBuilder` has run once; `CorpusIndex.getStatus().built` is
  `true`; subsequent ticks do not re-index. Questions answer without a manual
  build step.
