---
title: Curation Workflow Notes
summary: 'RLM curation workflow note: use recon precheck, prefer single-pass for small contexts, and verify curated file paths via result.applied[].filePath without readFile.'
tags: []
related: []
keywords: []
createdAt: '2026-05-20T10:41:56.145Z'
updatedAt: '2026-05-20T10:42:17.128Z'
---
## Reason
Curate newly provided workflow notes about RLM curation and verification

## Raw Concept
**Task:**
Document the curation workflow note for RLM-based context processing

**Changes:**
- Recon was already computed and should not be repeated.
- Single-pass processing is indicated for this compact context.
- Verification must use result.applied[].filePath rather than readFile.
- Use recon output to choose the processing mode
- Prefer single-pass curation for small contexts
- Verify curation via applied file paths

**Flow:**
recon -> choose mode -> extract or curate -> verify applied file paths

**Timestamp:** 2026-05-20T10:42:12.721Z

**Author:** ByteRover context engineering

## Narrative
### Structure
This note captures a compact RLM curation workflow and its verification rule for small context inputs.

### Dependencies
Depends on recon results and curate output metadata to confirm success.

### Highlights
The workflow emphasizes avoiding raw context output, using single-pass when suggested, and checking applied file paths for verification.

### Rules
Do NOT print raw context. Do NOT call tools.curation.recon. For chunked extraction use tools.curation.mapExtract(). Pass taskId as a bare variable. Verify via result.applied[].filePath.

## Facts
- **rlm_curation_mode**: For small contexts, recon may recommend single-pass curation. [convention]
- **mapextract_task_id**: When chunked extraction is needed, pass taskId as a bare variable to mapExtract. [convention]
- **verification_method**: Verification should use result.applied[].filePath and not readFile. [convention]
