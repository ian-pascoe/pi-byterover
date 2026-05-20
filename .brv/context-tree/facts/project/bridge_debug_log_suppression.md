---
title: Bridge Debug Log Suppression
summary: Debug console output should stay out of user-facing output; lack of logs does not mean message dispatch is blocked.
tags: []
related: []
keywords: []
createdAt: '2026-05-20T10:42:44.205Z'
updatedAt: '2026-05-20T10:42:44.205Z'
---
## Reason
Preserve guidance about suppressed debug output and UI notification handling

## Raw Concept
**Task:**
Document guidance for suppressing routine debug console output and interpreting UI/send behavior

**Changes:**
- Keep routine debug console output out of user-facing output
- Surface actionable warnings and errors via UI notification instead
- Treat hidden debug suppression separately from actual message dispatch flow

**Flow:**
debug console output -> suppression -> UI notification for actionable issues -> evaluate actual send flow when diagnosing

**Timestamp:** 2026-05-20T10:42:38.087Z

**Author:** ByteRover session notes

## Narrative
### Structure
This knowledge captures output-handling guidance and a diagnostic distinction between suppressed logs and true send failures.

### Dependencies
Applies when diagnosing UI/send issues in the extension or bridge flow.

### Highlights
Absence of logs does not imply blocked send; the actual message dispatch path must be considered separately.

### Rules
Keep routine debug console output out of user-facing output; surface actionable warnings and errors via UI notification instead.

### Examples
When UI/send issues are investigated, first determine whether logs are intentionally suppressed before concluding dispatch is broken.
