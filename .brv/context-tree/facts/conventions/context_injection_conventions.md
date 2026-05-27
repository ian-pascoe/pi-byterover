---
title: Context Injection Conventions
summary: RLM curation workflow uses precomputed recon, single-pass processing for small contexts, 300000ms timeout for mapExtract calls, and filePath-based verification.
tags: []
related: []
keywords: []
createdAt: '2026-05-27T10:03:47.950Z'
updatedAt: '2026-05-27T10:13:51.364Z'
---
## Reason
Curate RLM approach instructions and execution constraints for curation tasks

## Raw Concept
**Task:**
Document the RLM curation approach and execution constraints for context injection tasks

**Changes:**
- Clarified block layout for injected memory
- Added convention to keep safety text separate from upstream content
- Use precomputed recon when suggestedMode is single-pass
- Avoid calling tools.curation.recon again in this flow
- Apply 300000ms timeout on code_exec calls containing mapExtract
- Verify curation via result.applied[].filePath

**Files:**
- src/index.ts

**Flow:**
context variable -> precomputed recon -> single-pass extraction -> curate -> verify via applied file paths

**Timestamp:** 2026-05-27T10:13:36.117Z

**Author:** ByteRover context engineering workflow

## Narrative
### Structure
This note captures the operational pattern for RLM curation tasks that arrive with context, history, metadata, and task ID variables already injected.

### Dependencies
Depends on the precomputed recon result and the curation helper APIs; the flow assumes small contexts can be handled without chunking.

### Highlights
The workflow explicitly avoids redundant recon, uses single-pass processing when suggested, and enforces a 300000ms timeout for any code_exec containing mapExtract.

### Rules
Do NOT print raw context. Do NOT call tools.curation.recon when recon is already precomputed. Use result.applied[].filePath for verification and do not call readFile for verification.

### Examples
Example block shape: <byterover-context>
Security note: ...

Recalled ByteRover memory:
{content}
</byterover-context>

## Facts
- **rlm_curation_flow**: RLM curation tasks provide context, history, metadata, and task ID variables, and recon is precomputed for single-pass processing. [convention]
- **recon_usage**: For this task, tools.curation.recon should not be called because recon was already computed. [convention]
- **mapextract_timeout**: When mapExtract is used in code_exec, the code_exec timeout must be set to 300000 on the tool call itself. [convention]
- **verification_method**: Verification should use result.applied[].filePath and should not call readFile for verification. [convention]
