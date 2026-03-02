---
name: research
description: Web research deep dive on a specific topic using Firecrawl
triggers: [research, deep dive, find more, augment]
---

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

## Workflow

Execute `workflows/definitions/research-augment.yaml`

## Prerequisites

- Firecrawl MCP must be enabled in `state/config/tools.yaml`

## Output

- `output/{course-id}/notes/research-{topic}.md`
- Optionally: Notion page with tags
