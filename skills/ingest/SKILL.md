---
name: ingest
description: Scan and index course materials — build module configs, glossary, and retrieval indexes
triggers: [ingest, load course, scan materials, index]
---

## What This Does

Scans the materials directory for the active course and:
1. Identifies all files and maps them to modules
2. Normalizes folder structure (handles inconsistent naming)
3. Extracts key terms and builds the course glossary
4. Generates module config YAMLs from study guide content
5. Updates retrieval indexes (concepts, glossary, recent)

Uses incremental detection — skips already-indexed modules.

## Usage

```
/ingest              # Ingest active course
/ingest --force      # Re-ingest everything (ignore existing configs)
```

## Workflow

Execute `workflows/definitions/ingest-course.yaml`

## Prerequisites

- Course materials placed in `materials/{course-id}/`
- Active course set in `state/courses/_active.yaml`

## Output

- `courses/{course-id}/modules/*.yaml` — Module configurations
- `courses/{course-id}/glossary.yaml` — Course glossary
- Updated `indexes/concepts.yaml`, `indexes/glossary-index.yaml`, `indexes/recent.yaml`
