# Data model — job-posting-eeo

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PostingRequest` | `company` | `String` | no | Hiring company name. |
| | `roleTitle` | `String` | no | Role being advertised. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `WorkPlan` | `cultureBrief` | `String` | no | Instruction dispatched to CultureAnalyst. |
| | `roleBrief` | `String` | no | Instruction dispatched to RoleAnalyst. |
| `CultureProfile` | `values` | `List<String>` | no | 3–5 company value phrases. |
| | `tone` | `String` | no | One-word posting voice. |
| | `summary` | `String` | no | 2–3 sentence voice guidance. |
| | `analyzedAt` | `Instant` | no | When the CultureAnalyst completed. |
| `RoleSpec` | `responsibilities` | `List<String>` | no | 4–6 duties. |
| | `qualifications` | `List<String>` | no | 4–6 capability markers. |
| | `marketSalaryRange` | `String` | no | Seeded market range, e.g. `"$72k–$95k"`. |
| | `analyzedAt` | `Instant` | no | When the RoleAnalyst completed. |
| `DraftPosting` | `title` | `String` | no | Posting title. |
| | `body` | `String` | no | 120–200 word posting body. |
| | `eeoStatement` | `String` | no | Standard equal-opportunity clause. |
| | `guardrailVerdict` | `String` | no | `"ok"` or `"blocked: <reason>"`. |
| | `draftedAt` | `Instant` | no | When the draft completed. |
| `SanitizedPosting` | `title` | `String` | no | Sanitized title. |
| | `body` | `String` | no | Sanitized body (terms removed). |
| | `removedTerms` | `List<String>` | no | Protected-class terms the sanitizer removed (may be empty). |
| | `sanitizedAt` | `Instant` | no | When the sanitizer completed. |
| `JobPosting` (entity state) | `postingId` | `String` | no | Unique id. |
| | `company` | `String` | no | Hiring company. |
| | `roleTitle` | `String` | no | Role. |
| | `status` | `PostingStatus` | no | See enum. |
| | `culture` | `Optional<CultureProfile>` | yes | Populated after CultureAttached. |
| | `role` | `Optional<RoleSpec>` | yes | Populated after RoleAttached. |
| | `draft` | `Optional<DraftPosting>` | yes | Populated after PostingDrafted. |
| | `sanitized` | `Optional<SanitizedPosting>` | yes | Populated after PostingSanitized. |
| | `failureReason` | `Optional<String>` | yes | Populated on PostingBlocked / PostingDegraded. |
| | `createdAt` | `Instant` | no | When PostingCreated emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the posting reached a terminal state. |

## Enums

`PostingStatus`: `PLANNING`, `ANALYZING`, `DRAFTED`, `SANITIZED`, `CLEARED`, `BLOCKED`, `DEGRADED`.

## Events (`JobPostingEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PostingCreated` | `company, roleTitle, createdAt` | → PLANNING |
| `CultureAttached` | `culture` | (no status change; populates the field) |
| `RoleAttached` | `role` | (no status change; populates the field) |
| `PostingDrafted` | `draft` | → DRAFTED |
| `PostingBlocked` | `draft, failureReason` | → BLOCKED, `finishedAt = now` |
| `PostingDegraded` | `draft(partial), failureReason` | → DEGRADED, `finishedAt = now` |
| `PostingSanitized` | `sanitized` | → SANITIZED |
| `PostingCleared` | — | → CLEARED, `finishedAt = now` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `RequestSubmitted` | `postingId, company, roleTitle, requestedBy, submittedAt` |

## View row

`JobPostingRow` mirrors `JobPosting` minus heavy nested payloads (truncate `draft.body` and the nested analyses in the row to keep the SSE stream small). Every nullable lifecycle field is declared `Optional<T>` (Lesson 6). The UI fetches the full posting by id on click. The view exposes ONE query, `getAllPostings` (`SELECT * AS postings FROM posting_view`) — no `WHERE status` filter, because Akka cannot auto-index the enum column (Lesson 2); callers filter client-side.
