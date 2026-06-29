# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a multi-image batch and watch parallel synthesis

**Preconditions:** service running on `http://localhost:9850/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a question ("What objects are visible in each image?") and attach 3 images. Submit.
2. Observe the new job row via SSE.

**Expected:** the job progresses `QUEUED → ANALYSING → SYNTHESISED` within ~90 s. The expanded row shows three `ImageReport` cards (one per image), each with a description, tags, and detected labels. The `reports` arrive close together because the sub-agents ran in parallel. The synthesised answer references findings from all three reports.

## J2 — Sub-agent timeout produces a partial result

**Preconditions:** `HaikuImageAnalyst` step timeout set to 1 s (test override).

**Steps:**
1. Submit a 3-image batch.
2. Watch the job.

**Expected:** one or more image steps time out; the workflow routes to `partialStep`; the coordinator synthesises from whichever `ImageReport` objects arrived. The job enters `PARTIAL`; the expanded row shows only the reports that completed. The synthesised answer notes how many images were not analysed. No infinite retry.

## J3 — PII is redacted before sub-agents see the image

**Preconditions:** an image payload that contains a face (a JPEG where the heuristic triggers `piiDetected = true`).

**Steps:**
1. Submit a batch that includes the face-containing image.
2. Let the job complete to `SYNTHESISED`.

**Expected:** the `AnalysisJobRow.sanitizedImages` entry for that image shows `piiDetected = true`. The corresponding `ImageReport` description does not mention a face or any person. The synthesised answer does not describe facial features. The job is not blocked unless the coordinator incorrectly references the redacted content.

## J4 — Guardrail blocks a synthesis that references redacted content

**Preconditions:** mock coordinator configured to return a `SynthesisedAnswer` whose `answer` text contains the string "face" when an image with `piiDetected = true` is present (test fixture).

**Steps:**
1. Submit a batch with one face-containing image and the mock coordinator fixture active.
2. Watch the job.

**Expected:** `guardrailStep` detects the PII bleed-through; the workflow calls `block`; the job enters `BLOCKED` with a `failureReason`. The `SynthesisedAnswer` is never surfaced in the App UI's expanded row.

## J5 — Eval score appears beside a synthesised job

**Preconditions:** at least one `SYNTHESISED` job without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually via a test endpoint.
2. Observe the job row.

**Expected:** the job gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score badge. Delivery of the original synthesised answer was never held up by the eval (non-blocking).

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for at least 2 minutes.

**Expected:** `BatchSimulator` drips a canned 3-image batch from `image-batches.jsonl` every 90 s; each becomes a job that flows through the full pipeline. The App UI is non-empty on first load without any manual submission.
