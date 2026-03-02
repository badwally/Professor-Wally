---
name: setup
description: First-run configuration — create learner profile, set course objectives, configure tool integrations
triggers: [setup, configure, first run, get started]
---

## What This Does

Runs the first-time setup wizard that:
1. Builds your "how-i-learn" profile through guided questions
2. Sets your course objectives and definitions of success
3. Configures MCP integrations (Notion, Gmail, Firecrawl)
4. Initializes all state files

## Workflow

Execute `workflows/definitions/setup.yaml`

## Prerequisites

None — this is the entry point.

## Output

- `state/config/learner.yaml` — Your learner profile
- `state/config/tools.yaml` — Enabled integrations
- `state/courses/_active.yaml` — Active course pointer
- Initialized progress, SRS, and memory files
