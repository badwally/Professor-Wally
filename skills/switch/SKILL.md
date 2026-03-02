---
name: switch
description: Switch the active course
triggers: [switch, change course, load course]
---

## What This Does

Switches the active course to a different one:
1. Validates the course config exists at `courses/{course-id}/course.yaml`
2. Updates `state/courses/_active.yaml`
3. Loads the new course's progress
4. Reports current status in the new course

## Usage

```
/switch mit-ai-xpro      # Switch to MIT xPRO course
/switch agentic-ai-vp    # Switch to Agentic AI VP curriculum
```

## Workflow

Inline execution (no separate workflow file)

## Prerequisites

- Course must exist in `courses/{course-id}/`
- Course should be ingested (`/ingest`) for best experience

## Output

- Updated `state/courses/_active.yaml`
- Progress summary for the new active course
