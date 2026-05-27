---
title: Curation Workflow Notes
summary: ByteRover project curation workflow notes covering recon-first processing, single-pass handling for small contexts, and verification requirements.
tags: []
related: [facts/project/context_curation_workflow_notes.md]
keywords: []
createdAt: '2026-05-20T10:41:56.145Z'
updatedAt: '2026-05-27T10:06:32.372Z'
---
## Reason
Curate project-specific curation workflow facts from RLM context

## Raw Concept
**Task:**
Document the RLM curation workflow requirements for this context task

**Changes:**
- Recon was already computed and should not be repeated.
- Single-pass processing is indicated for this compact context.
- Verification must use result.applied[].filePath rather than readFile.
- Use recon output to choose the processing mode
- Prefer single-pass curation for small contexts
- Verify curation via applied file paths
- Recognized precomputed recon metadata for a small single-pass context
- Captured timeout and taskId requirements for mapExtract usage
- Recorded verification guidance that avoids readFile

**Flow:**
recon -> single-pass extraction -> curate -> verify via applied file paths

**Timestamp:** 2026-05-27T10:06:22.667Z

**Author:** ByteRover context engineer

## Narrative
### Structure
This knowledge belongs in the project facts domain because it describes operational curation behavior rather than product functionality.

### Dependencies
The workflow depends on precomputed recon metadata and the curation tools available in the sandbox environment.

### Highlights
Small contexts should be handled in a single pass, and mapExtract-based chunking has explicit timeout and taskId handling rules.

### Rules
Do NOT print raw context. Do NOT call tools.curation.recon when recon is already precomputed. Use tools.curation.groupBySubject() and tools.curation.dedup() to organize extractions when chunking is required. Verify via result.applied[].filePath and do NOT call readFile for verification.

## Facts
- **curation_recon_mode**: The recon step was already computed for this curation task, and suggestedMode is single-pass with suggestedChunkCount 1 for a 1251-character, 29-line context. [project]
- **mapextract_timeout_requirement**: For chunked extraction, mapExtract must receive taskId as a bare variable and code_exec containing mapExtract must use timeout 300000 on the tool call itself. [project]
- **curation_verification_method**: Verification of curation should use result.applied[].filePath rather than calling readFile. [project]
- **raw_context_handling**: The RLM curation approach requires avoiding raw context printing and using concise summaries instead. [project]
