# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a batch and watch parallel processing

**Preconditions:** service running on `http://localhost:9593/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter two raw document strings and Submit.
2. Observe the batch progress bar and document rows via SSE.

**Expected:** both documents progress `QUEUED → PROCESSING → PROCESSED` within ~90 s. The batch status moves `PENDING → IN_PROGRESS → COMPLETE`. Each expanded document row shows `ExtractedFields` (date, author, referenceNumber, bodyText), a `DocumentClassification` (risk category + flagged topics), and a `sanitizedText`. Extraction and classification arrive close together because the workers ran in parallel.

## J2 — Extractor timeout degrades a document and the batch finishes PARTIAL

**Preconditions:** `Extractor` step timeout set to 1 s (test override).

**Steps:**
1. Submit a two-document batch.
2. Watch both document rows and the batch progress bar.

**Expected:** the `extractStep` times out for at least one document; the workflow routes to `degradeStep`; that document enters `DEGRADED` with a `failureReason`. The batch still processes the remaining document and finishes `PARTIAL`. No infinite retry.

## J3 — PII in a document is redacted before persistence

**Preconditions:** one of the submitted documents contains a synthetic SSN (e.g., "SSN: 123-45-6789") in its body text.

**Steps:**
1. Submit the document.
2. Wait for the document to reach `PROCESSED`.
3. Expand the document row.

**Expected:** the `sanitizedText` field contains `[REDACTED:national-id]` in place of the SSN. The `piiRedactions` list shows one entry: `{ "fieldName": "bodyText", "piiType": "national-id", "replacement": "[REDACTED:national-id]" }`. The raw SSN value does not appear in any persisted field.

## J4 — Quality score appears beside a processed document

**Preconditions:** at least one `PROCESSED` document without a `qualityScore`.

**Steps:**
1. Wait for `QualitySampler` to run (every 5 minutes), or trigger it directly.
2. Observe the document row in the App UI.

**Expected:** the document gains a `qualityScore` (1–5) and a `qualityRationale`; the App UI row shows the score pill. Document delivery was never blocked by the quality eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for at least 3 minutes.

**Expected:** `DocumentSimulator` drips a sample batch from `sample-batches.jsonl` every 90 s; each batch fans out through the full pipeline. The App UI shows a non-empty document list on first load. The batch progress bar reflects each simulator-submitted batch completing.
