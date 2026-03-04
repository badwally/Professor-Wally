---
name: research
description: Web research deep dive on a specific topic using Firecrawl
user_invocable: true
---

# /research — Web Research Deep Dive

Execute the research workflow defined in `workflows/definitions/research-augment.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/research-augment.yaml`
2. Read policies listed in the workflow's `context.policies` from `policies/`
3. Load the active course from `state/courses/_active.yaml`
4. Check that Firecrawl is enabled in `state/config/tools.yaml`
5. Parse the topic from the user's argument (e.g., `/research "gradient descent"`)
6. Execute the workflow steps in order

## What This Does

Uses Firecrawl to research a topic in depth:
1. Searches for 3-5 authoritative, current sources
2. Extracts relevant content and key insights
3. Synthesizes findings into a research note
4. Saves to `output/{course-id}/notes/research-{topic}.md`
5. Optionally syncs to Notion

## Usage

```
/research "gradient descent optimization"
/research "Delta Model applications in SaaS"
```

## Prerequisites

- Firecrawl MCP must be enabled in `state/config/tools.yaml`

## Output

- `output/{course-id}/notes/research-{topic}.md`
- Optionally: Notion page with tags
