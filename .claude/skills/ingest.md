---
name: ingest
description: Scan and index course materials — build module configs, glossary, and retrieval indexes
user_invocable: true
---

# /ingest — Scan and Index Course Materials

Execute the ingest workflow defined in `workflows/definitions/ingest-course.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/ingest-course.yaml`
2. Read policies listed in the workflow's `context.policies` from `policies/`
3. Load the active course from `state/courses/_active.yaml`
4. Execute the workflow steps in order, respecting all gates

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

## Prerequisites

- Course materials placed in `materials/{course-id}/`
- Active course set in `state/courses/_active.yaml`

## Output

- `courses/{course-id}/modules/*.yaml` — Module configurations
- `courses/{course-id}/glossary.yaml` — Course glossary
- Updated `indexes/concepts.yaml`, `indexes/glossary-index.yaml`, `indexes/recent.yaml`
