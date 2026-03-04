---
name: enrich
description: Generate enriched study guide and question banks for a module
user_invocable: true
---

# /enrich — Generate Enriched Study Guide

Execute the enrich workflow defined in `workflows/definitions/enrich-module.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/enrich-module.yaml`
2. Read policies listed in the workflow's `context.policies` from `policies/`
3. Load the active course from `state/courses/_active.yaml`
4. Parse the module number from the user's argument (e.g., `/enrich 1` means Module 1)
5. Execute the workflow steps in order, respecting all gates

## What This Does

1. Loads module config and all source materials
2. (Optional) Searches Substack via Gmail for supplementary material
3. (Optional) Researches current sources via Firecrawl
4. Generates an enriched, comprehensive study guide
5. Generates study questions (15-20, Bloom-distributed)
6. Generates mastery questions (8-12, ascending difficulty)
7. Seeds the SRS queue with new items
8. (Optional) Syncs to Notion with tags

Skips already-enriched modules unless `--force` is used.

## Usage

```
/enrich 1            # Enrich Module 1
/enrich 3 --force    # Re-enrich Module 3 even if already done
```

## Prerequisites

- Course must be ingested (`/ingest`)
- Module materials must exist

## Output

- `output/{course-id}/study-guides/module-{nn}-study-guide.md`
- `output/{course-id}/questions/module-{nn}-study-questions.md`
- `output/{course-id}/questions/module-{nn}-mastery-questions.md`
- Updated `state/progress/{course-id}/spaced-repetition.yaml`
