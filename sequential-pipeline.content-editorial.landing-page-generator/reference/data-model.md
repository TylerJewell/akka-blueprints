# Data model — Landing Page Generator

Every record the generated system defines, with nullability. Lifecycle fields that are unset until their event fires are `Optional<T>` (Lesson 6).

## `LandingPage` (entity state + view row type)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Page id (UUID) |
| `concept` | `Optional<String>` | yes | The submitted concept |
| `status` | `PageStatus` | no | Lifecycle state |
| `headline` | `Optional<String>` | yes | Set on `CopyDrafted` |
| `subhead` | `Optional<String>` | yes | Set on `CopyDrafted` |
| `bodyCopy` | `Optional<String>` | yes | Set on `CopyDrafted` |
| `sections` | `Optional<List<String>>` | yes | Set on `PageStructured` |
| `ctaPrimary` | `Optional<String>` | yes | Set on `CtaWritten` |
| `ctaSecondary` | `Optional<String>` | yes | Set on `CtaWritten` |
| `reviewScore` | `Optional<Integer>` | yes | Set on `PageMarkedReady` |
| `flagReason` | `Optional<String>` | yes | Set on `PageFlagged` |
| `copyDraftedAt` | `Optional<String>` | yes | ISO-8601, set on `CopyDrafted` |
| `structuredAt` | `Optional<String>` | yes | ISO-8601, set on `PageStructured` |
| `ctaWrittenAt` | `Optional<String>` | yes | ISO-8601, set on `CtaWritten` |
| `reviewedAt` | `Optional<String>` | yes | ISO-8601, set on `PageMarkedReady`/`PageFlagged` |

`emptyState()` returns `LandingPage.initial("")` — no `commandContext()` reference (Lesson 3).

## Status enum

```java
public enum PageStatus { DRAFTING_COPY, STRUCTURING, WRITING_CTA, REVIEWING, READY, FLAGGED }
```

## Agent result records

| Record | Fields | Returned by |
|---|---|---|
| `CopyDraft` | `String headline, String subhead, String bodyCopy` | `CopyAgent` |
| `PageSections` | `List<String> sections` | `StructureAgent` |
| `CtaBlocks` | `String primary, String secondary` | `CtaAgent` |
| `ReviewResult` | `boolean passed, int score, String reason` | review step (G1 guardrail) |

## Events (on `LandingPageEntity`)

| Event | Trigger |
|---|---|
| `CopyDrafted` | `recordCopy` after `CopyAgent` returns |
| `PageStructured` | `recordStructure` after `StructureAgent` returns |
| `CtaWritten` | `recordCta` after `CtaAgent` returns |
| `PageMarkedReady` | `markReady` when the brand-safety review passes |
| `PageFlagged` | `markFlagged` when the brand-safety review fails |

## `ConceptQueue` events

| Event | Trigger |
|---|---|
| `ConceptQueued` | `enqueueConcept` from the endpoint or the simulator |

## View

`LandingPagesView` — row type `LandingPage`, one row per page. One query: `getAllPages` (`SELECT * AS pages FROM pages_view`). No `WHERE status` filter — Akka cannot auto-index enum columns (Lesson 2); callers filter client-side.
