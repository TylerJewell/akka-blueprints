# User journeys — sequential-cv-tailor

## J1 — Submit a candidate and posting, receive a tailored CV

**Preconditions:** Service running on declared port (`http://localhost:9856/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded candidate `alice-chen` and posting `senior-java-engineer` fixtures are present in `src/main/resources/sample-data/`.

**Steps:**
1. Open `http://localhost:9856/` → App UI tab.
2. From the **Candidate** dropdown, pick `Alice Chen`.
3. From the **Job Posting** dropdown, pick `Senior Java Engineer`.
4. Click **Tailor CV**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `GENERATING` within 1 s more.
- Within ~20 s the card reaches `GENERATED`. The right pane shows Stage 1 (Base CV) with a non-empty `generatedSummary`, ≥ 3 skills, and ≥ 1 experience entry with non-empty bullet points.
- Within ~20 s more the card reaches `TAILORED`. Stage 2 shows the tailored summary with at least one required-keyword chip shown in green.
- Within ~2 s the card reaches `SCORED`. The alignment score chip shows ≥ 3. The keyword-match section lists at least one covered keyword.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — PII does not reach the model

**Preconditions:** Service running. Any model provider (the sanitizer runs regardless). `LOG_LEVEL=DEBUG` so model-call payloads are logged.

**Steps:**
1. Submit any seeded candidate against any seeded posting.
2. Wait for `SCORED`.
3. Inspect the service log for model-call lines. Filter to entries tagged with the requestId.

**Expected:**
- No log line from any model call contains the strings `"candidateName"` (with a non-redacted value), `"email"`, `"phone"`, `"address"`, or `"dateOfBirth"` — each of those field names appears only with value `"[REDACTED]"` in model-call payloads.
- The entity's `sanitizationCount` is ≥ 1 (at least the generator call fired the sanitizer).
- The sanitization-audit strip in the UI shows at least one event with `agent = CvGeneratorAgent` and `removedCount ≥ 1`.
- The tailor model-call payload contains only `BaseCv` and `JobPosting` fields; no `CandidateProfile` wrapper appears.

## J3 — Missing required keywords produce a low alignment score

**Preconditions:** Mock LLM mode. The mock's `tailor-cv.json` includes one entry where all `requiredKeywords` from the posting are absent from the `tailoredSummary` and experience bullets.

**Steps:**
1. Submit the seeded candidate and posting combination that maps to the zero-keyword mock entry (the mock selects it on the 4th request modulo `seedFor(requestId)`).
2. Watch for a card whose border highlights red.

**Expected:**
- The `TailoredCv` is well-formed (the sanitizer only checks PII, not content).
- The alignment score chip shows **1** or **2** and the rationale reads: `"Required keywords missing: <keyword list>."`.
- The card border highlights red. The Keywords-missed section in the right pane lists all required keywords.
- The other requests in the run with the happy-path mock entry scored ≥ 3.

## J4 — Raw candidate profile is not forwarded to the tailor stage

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded candidate.
2. Wait for `SCORED`.
3. Inspect the service log for the tailor agent's task-start line. Check what instruction context it received.

**Expected:**
- The tailor task's instruction context contains a `BaseCv` JSON structure and a `JobPosting` JSON structure.
- The instruction context does not contain a `CandidateProfile` wrapper or the raw PII fields (`email`, `phone`, `address`, `dateOfBirth`) with real values.
- The workflow log shows `tailorStep` reading the `BaseCv` from `CvRequestEntity` and building the instruction context from it — not from the original submission payload.

## J5 — Custom posting with no matching skills produces an honest result

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Use the API directly: `POST /api/cv-requests` with candidate `bob-ramirez` (data science background) and posting `senior-java-engineer` (requires Java, Spring Boot, Kubernetes — none of which Bob's profile mentions).
2. Wait for `SCORED`.

**Expected:**
- `CvGeneratorAgent` produces a `BaseCv` from Bob's data science profile (Python, scikit-learn, SQL skills).
- `CvTailorAgent` returns a `TailoredCv` whose `keywordMatches` show all three required Java keywords with `sourceSection = null`.
- `AlignmentScorer` emits score 1 with rationale naming all three missing required keywords.
- The pipeline completes without error. Nothing crashes. The honest mismatch result is surfaced to the recruiter.

## J6 — Concurrent requests are isolated

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit three CV requests in rapid succession (all three seeded candidate/posting pairs).
2. Watch the live list until all three reach `SCORED`.

**Expected:**
- Each request transitions independently; no two requests share state.
- Each request's entity has its own `sanitizationCount` reflecting only its own model calls.
- The alignment score for each request reflects only its own `TailoredCv` and `JobPosting`.
- No cross-request data appears in any card's detail pane.
