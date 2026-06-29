# Data model — screenplay-writer

Authoritative record set. `/akka:implement` writes these exactly.

## ScreenplayJob (entity state and view row)

Lifecycle fields are `Optional<T>` (Lesson 6). List fields default to empty, not null.

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Job id (the workflow id) |
| `sourceLabel` | `String` | no | Short label for the source thread |
| `status` | `ScreenplayStatus` | no | Lifecycle state |
| `sanitizedText` | `Optional<String>` | yes | Source thread after PII redaction |
| `redactionCount` | `Optional<Integer>` | yes | Number of redactions the sanitizer made |
| `characters` | `List<String>` | no (empty until ANALYZED) | Character labels from the analyzer |
| `setting` | `Optional<String>` | yes | Scene setting |
| `synopsis` | `Optional<String>` | yes | Short synopsis |
| `tone` | `Optional<String>` | yes | Emotional register |
| `dialogue` | `Optional<String>` | yes | Draft scene dialogue |
| `screenplay` | `Optional<String>` | yes | Final formatted screenplay |
| `piiFindings` | `List<String>` | no (empty unless BLOCKED) | Residual PII flagged by the guardrail |
| `sanitizedAt` | `Optional<Instant>` | yes | When sanitization completed |
| `analyzedAt` | `Optional<Instant>` | yes | When analysis completed |
| `dialogueAt` | `Optional<Instant>` | yes | When dialogue was written |
| `completedAt` | `Optional<Instant>` | yes | When the screenplay completed |
| `failureReason` | `Optional<String>` | yes | Reason on FAILED |

`emptyState()` returns `ScreenplayJob.initial("", "")` with empty lists and absent Optionals — no `commandContext()` reference (Lesson 3).

## ScreenplayStatus enum

`RECEIVED`, `SANITIZED`, `ANALYZED`, `DIALOGUE_WRITTEN`, `COMPLETED`, `BLOCKED`, `FAILED`.

The view row stores `status.name()` as a `String` column so list filtering works without enum indexing (Lesson 2).

## Events (ScreenplayEntity)

| Event | Trigger |
|---|---|
| `ThreadReceived` | Job created from a queued or submitted thread |
| `ThreadSanitized` | `sanitizeStep` finished; carries sanitized text and redaction count |
| `ThreadAnalyzed` | `analyzeStep` returned a `ThreadAnalysis` |
| `DialogueWritten` | `dialogueStep` returned a `DialogueDraft` |
| `ScreenplayCompleted` | `formatStep` returned a clean `Screenplay` |
| `ScreenplayBlocked` | `formatStep` guardrail found residual PII |
| `JobFailed` | A working step exhausted retries |

## InboundThreadQueue

| Event | Trigger |
|---|---|
| `ThreadQueued` | `enqueueThread(sourceLabel, rawText)` from the endpoint or simulator |

## Agent result records

- `ThreadAnalysis(List<String> characters, String setting, String synopsis, String tone)`
- `DialogueDraft(String dialogue)`
- `Screenplay(String formattedText, boolean containsPii, List<String> piiFindings)`
