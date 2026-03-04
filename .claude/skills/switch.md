---
name: switch
description: Switch the active course
user_invocable: true
---

# /switch — Switch Active Course

This is an inline skill (no separate workflow file). Execute directly by updating active course state.

## Instructions

1. Parse the course-id from the user's argument (e.g., `/switch mit-ai-xpro`)
2. Validate that `courses/{course-id}/course.yaml` exists
3. Update `state/courses/_active.yaml` to point to the new course
4. Load `state/progress/{course-id}/progress.yaml` for the new course
5. Report current status: module progress, overall score, SRS items due

## Usage

```
/switch mit-ai-xpro      # Switch to MIT xPRO course
/switch agentic-ai-vp    # Switch to Agentic AI VP curriculum
```

## Prerequisites

- Course must exist in `courses/{course-id}/`
- Course should be ingested (`/ingest`) for best experience

## Output

- Updated `state/courses/_active.yaml`
- Progress summary for the new active course
