---
name: setup
description: First-run configuration — create learner profile, set course objectives, configure tool integrations
user_invocable: true
---

# /setup — First-Run Setup Wizard

Execute the setup workflow defined in `workflows/definitions/setup.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/setup.yaml`
2. Read the policies listed in the workflow's `context.policies` from `policies/`
3. Execute each step in order:
   - **welcome**: Display the welcome message. Wait for user confirmation before proceeding.
   - **setup-learning-style**: Ask the learning profile questions conversationally. Read `state/config/learner.template.yaml` for the schema. Write the learner profile to `state/config/learner.yaml`. Present the file to the user for review/editing before proceeding.
   - **setup-course-objectives**: Ask course objective questions. Read `courses/_template/course.template.yaml` for the schema. Write course config to `courses/{course-id}/course.yaml`. Present for review/editing.
   - **setup-success-criteria**: Ask success criteria questions. Update the course config. Present for review/editing.
   - **setup-tools**: Ask about MCP integrations (Notion, Gmail, Firecrawl). Read `state/config/tools.template.yaml` for the schema. Write to `state/config/tools.yaml`. Present for review/editing.
   - **initialize-state**: Create all state files: `state/courses/_active.yaml`, `state/progress/{course-id}/progress.yaml`, `state/progress/{course-id}/spaced-repetition.yaml`, `state/progress/{course-id}/session-log.yaml`, `state/memory/memory.yaml`.
   - **setup-summary**: Display a concise summary of the setup and suggest next steps (`/ingest`, `/enrich`, `/study`).

4. At each `gate` step, pause and wait for the user before continuing:
   - `confirm` gates: ask the user to confirm before proceeding
   - `edit` gates: present the generated content and let the user review/modify
   - `display` gates: show the output

## Key Rules

- Gather answers conversationally — do not dump all questions at once
- Adapt phrasing to the learner's role and context
- All MCP integrations are optional; the system works without them
- Use the template files for output schemas
- Set default scoring thresholds: passing 0.70, mastery 0.90, fluency 0.80
