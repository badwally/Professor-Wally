---
id: context-discipline
scope: always-on
derived_from: chief-of-staff P18
---

## Rule

Minimize context window consumption through targeted, efficient information retrieval and generation.

### Reading Materials
- Read study guides section-by-section, not entire files at once
- Use module configs and indexes to locate specific content
- Extract only the relevant sections when enriching or tutoring

### Generating Content
- Study guides should be comprehensive but not redundant with source material
- Questions should be self-contained — include enough context to answer without re-reading the guide
- Session summaries should capture decisions and gaps, not transcribe the entire conversation

### State Management
- Load only the active course's progress, not all courses
- Memory entries should be concise (1-2 sentences) with clear scope tags
- Prune session logs older than 30 days to summaries only

### Workflow Execution
- Optional workflow steps should check tool availability before attempting
- If a step fails, log the failure and continue — don't retry in the same session
- Chain outputs between steps to avoid re-reading source files

## Exceptions

- During `/ingest`, reading full study guides is necessary for glossary extraction
- During initial setup, reading template files fully is expected
