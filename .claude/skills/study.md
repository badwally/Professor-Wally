---
name: study
description: Interactive guided study session — the main learning loop
user_invocable: true
---

# /study — Interactive Guided Study Session

Execute the study workflow defined in `workflows/definitions/study-session.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/study-session.yaml`
2. Read policies listed in the workflow's `context.policies` from `policies/`
3. Load the active course from `state/courses/_active.yaml`
4. Parse arguments: module number, optional `--section` flag
5. Check for session checkpoint in `state/progress/{course-id}/session-log.yaml` — offer to resume if interrupted
6. Execute the workflow steps in order, respecting all gates

## What This Does

Walks through the enriched study guide section by section:

1. **Assess state**: Checks progress, SRS queue, last session checkpoint
2. **SRS review**: If items are due, starts with spaced repetition warm-up
3. **Section-by-section study**: For each section:
   - Presents the section digest
   - Active recall: asks you to explain key concepts first
   - Study questions with immediate scoring and feedback
   - Mastery questions (progressive depth) with rubric-scored evaluation
   - Section score displayed
4. **Module score**: Composite score computed, gaps identified
5. **Checkpoint**: Session state saved for resumption if interrupted
6. **Decision**: Below 0.80 = re-study weak areas; at 0.80+ = advance

## Usage

```
/study              # Continue where you left off
/study 2            # Study Module 2
/study 1 --section 3  # Resume Module 1 at section 3
```

## Prerequisites

- Module must be enriched (`/enrich <module>`)

## Output

- Updated `state/progress/{course-id}/progress.yaml`
- Updated `state/progress/{course-id}/spaced-repetition.yaml`
- Updated `state/progress/{course-id}/session-log.yaml`
- `output/{course-id}/session-summaries/{date}-session.md`
