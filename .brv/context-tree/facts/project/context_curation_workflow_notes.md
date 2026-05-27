---
title: Context Curation Workflow Notes
summary: Project curation notes covering RLM workflow expectations, verification requirements, and single-pass processing for small contexts
tags: []
related: [facts/project/pi_extension_output_and_diagnostics.md, facts/project/reasoning_effort_and_git_state.md, facts/project/curation_workflow_notes.md]
keywords: []
createdAt: '2026-05-20T10:37:28.836Z'
updatedAt: '2026-05-20T10:37:28.836Z'
---
## Reason
Curate concise operational notes from the provided context

## Raw Concept
**Task:**
Document the RLM curation approach and execution constraints for small contexts

**Changes:**
- Confirmed single-pass processing for a 388-character context
- Recorded taskId passing requirement for mapExtract
- Recorded timeout requirement for mapExtract-containing code_exec calls
- Recorded verification requirement using applied file paths

**Flow:**
recon -> single-pass extraction -> curate -> verify applied file paths

**Timestamp:** 2026-05-20T10:37:22.233Z

**Author:** ByteRover

## Narrative
### Structure
This note captures the operational procedure for curating a small context with the RLM approach, including when to skip chunking and how to validate applied results.

### Dependencies
Depends on the precomputed recon result and the curation execution environment.

### Highlights
Single-pass is recommended for the provided context size; mapExtract-specific calls require a 300000 ms tool timeout.

### Rules
Do NOT print raw context. Do NOT call tools.curation.recon when it has already been precomputed. Use tools.curation.groupBySubject() and tools.curation.dedup() to organize extractions. Verify via result.applied[].filePath and do NOT call readFile for verification.

## Facts
- **curation_mode**: For small contexts, the recommended mode is single-pass. [project]
- **task_id_passing**: When mapExtract is used, the taskId must be passed as a bare variable, not a string. [project]
- **mapextract_timeout**: Any code_exec call containing mapExtract must use timeout 300000 on the code_exec tool call itself. [project]
- **verification_method**: Verification should use result.applied[].filePath and should not call readFile for verification. [project]
