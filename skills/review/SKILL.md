---
name: review
description: Quick spaced repetition review of due items
triggers: [review, spaced repetition, srs, flashcards]
---

## What This Does

A focused SRS review session — no new material, just retention:
1. Loads items from `spaced-repetition.yaml` where `next_review <= today`
2. Presents each item's "front" (question/prompt)
3. Waits for your response
4. Reveals the "back" (answer)
5. You self-rate recall quality (0-5 scale)
6. Updates intervals using SM2 algorithm
7. Reports: items reviewed, average quality, retention rate

## Usage

```
/review             # Review all due items for active course
```

## Workflow

Inline execution (no separate workflow file — reads and updates SRS state directly)

## Output

- Updated `state/progress/{course-id}/spaced-repetition.yaml`
- Review summary displayed
