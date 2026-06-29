# User journeys — video-to-blog-pipeline

## J1 — Submit a URL and get a blog post

**Preconditions:** Service running on declared port (`http://localhost:9242/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded URL `https://www.youtube.com/watch?v=seed-akka-agents` has a matching `src/main/resources/sample-data/transcripts/seed-akka-agents.json` file.

**Steps:**
1. Open `http://localhost:9242/` → App UI tab.
2. From the **Pick a seeded URL** dropdown, pick `https://www.youtube.com/watch?v=seed-akka-agents`.
3. Click **Convert to blog post**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `TRANSCRIBING` within 1 s more.
- Within ~20 s the card reaches `TRANSCRIBED`. The right pane shows the Transcript panel: a text preview, the duration in seconds, and ≥ 2 chapters each with a non-empty title, `startSeconds`, and `endSeconds`.
- Within ~20 s more the card reaches `SUMMARISED`. The right pane shows ≥ 2 key points and a section outline with ≥ 2 entries; every `keyPoint.sourceStartSeconds` matches a chapter's `startSeconds` range from the transcript panel.
- Within ~20 s more the card reaches `DRAFTED`. The right pane shows an introduction paragraph and ≥ 2 section bodies; each body is at least 2 sentences.
- Within ~20 s more the card reaches `POLISHED`, then `EVALUATED` within 1 s of that. The right pane shows the Published-post panel with a title, polished introduction, per-section blocks, and a conclusion. The eval score chip shows 5/5 and the rationale reads "Section parity, heading completeness, word count, and body completeness all satisfied."
- Total elapsed time: ≤ 90 s on the happy path.

## J2 — Publish guardrail blocks a prohibited-content response

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `polish-post.json` includes one entry whose text contains a prohibited phrase from `prohibited-phrases.txt` — this is the deliberately prohibited-content entry.

**Steps:**
1. Submit any seeded URL three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/posts/sse`).

**Expected:**
- On the third submission's `polishStep`, the agent's first iteration produces a response containing a prohibited phrase. `PublishGuardrail` rejects it; a `GuardrailRejected{phase: "POLISH", reason: "content-violation: response contains prohibited phrase '...'"}` event lands on the entity.
- The prohibited response NEVER becomes the recorded `PostPolished` event — there is no log line from `BlogPostEntity.recordPost` for that iteration.
- The agent's second iteration produces clean content. The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected response with its full structured reason.

## J3 — Section-parity drift flags eval score

**Preconditions:** Mock LLM mode. The mock's `draft-post.json` and `polish-post.json` for one pairing include a draft with 3 sections and a polish result with 2 sections (one section was silently collapsed during polishing).

**Steps:**
1. Submit any seeded URL six times. (The section-parity-drift entry is selected once in every six runs by the mock's `seedFor(postId)` modulo.)
2. Watch the live list until a post's card border highlights red.

**Expected:**
- The flagged post lands `EVALUATED` with a well-formed `BlogPost` structure (the guardrail only checks content, not section count).
- The eval score chip shows **4** (section-parity check fails; the other three checks pass) and the rationale reads: *"Section parity failed: post has 2 sections but draft had 3."*
- The card's border highlights red. The editor knows to review this post before scheduling.
- The other five posts in the run scored 5/5.

## J4 — Per-task tool-call isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so each tool call is logged. Any model provider (real or mock — the guardrail runs either way).

**Steps:**
1. Submit any seeded URL.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the postId.

**Expected:**
- The TRANSCRIPT task's log entries show only `fetchTranscript` and `extractChapters` calls.
- The SUMMARISE task's log entries show only `extractKeyPoints` and `outlineSections` calls.
- The DRAFT task's log entries show only `writeSectionBody` and `composeIntroduction` calls.
- The POLISH task's log entries show only `polishProse` and `writeConclusion` calls.
- No cross-phase calls appear.
- The order of tasks in the log is TRANSCRIPT → SUMMARISE → DRAFT → POLISH. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.

## J5 — Word count out of range flags eval score

**Preconditions:** Service running. Mock LLM mode. The mock's `polish-post.json` includes one entry whose polished `BlogPost` has a `wordCount` of 50 (far below the 400-word floor).

**Steps:**
1. Submit seeded URLs until the mock selects the short-post entry (approximately once every six runs).
2. Watch for the card whose eval score chip shows ≤ 3.

**Expected:**
- The post lands `EVALUATED`. The Published-post panel shows the short post with all sections present and headings non-empty.
- The eval score chip shows **4** (word-count check fails; the other three checks pass) and the rationale reads: *"Word count out of range: post has 50 words, target is 400–1200."*
- The card's border highlights red. The editor sees immediately that the post is too short to publish.

## J6 — URL with no matching transcript file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/transcripts/<slug>.json` exists for the submitted URL.

**Steps:**
1. In the App UI, paste a custom URL (e.g., `https://www.youtube.com/watch?v=no-such-video`) into the URL field.
2. Click **Convert to blog post**.

**Expected:**
- `TranscriptTools.fetchTranscript` returns a `Transcript` with `text = ""` and `chapters = []`.
- The SUMMARISE task returns a `VideoSummary` with `keyPoints = []` and `sections = []`.
- The DRAFT task returns a `BlogDraft` with `introduction = "(no sections to draft)"` and `sections = []` (the agent's refusal behaviour per `prompts/blog-agent.md`).
- The POLISH task returns a `BlogPost` with `sections = []` and a short `conclusion`.
- The eval score chip shows **1** (all four structural rules fail for an empty post). The rationale reads: *"Section parity, heading completeness, word count, and body completeness all failed: post has 0 sections."*
- The pipeline completes; nothing crashes; the empty post is honestly empty.
