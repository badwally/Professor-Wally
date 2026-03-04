---
name: review
description: Quick spaced repetition review of due items
user_invocable: true
---

# /review — Spaced Repetition Review

This is an inline skill (no separate workflow file). Execute directly by reading and updating SRS state.

## Instructions

1. Load the active course from `state/courses/_active.yaml`
2. Read `state/progress/{course-id}/spaced-repetition.yaml`
3. Filter items where `next_review <= today`
4. For each due item:
   - Present the item's "front" (question/prompt)
   - Wait for the learner's response
   - Reveal the "back" (answer)
   - Ask the learner to self-rate recall quality (0-5 scale)
   - Update the item's interval using SM2 algorithm (see `policies/spaced-repetition.md`)
5. After all items reviewed, report: items reviewed, average quality, retention rate
6. Write updated SRS data back to `state/progress/{course-id}/spaced-repetition.yaml`

## SM2 Quality Scale

- 0: Complete blackout
- 1: Incorrect, but recognized the answer
- 2: Incorrect, but answer seemed easy to recall
- 3: Correct with serious difficulty
- 4: Correct after hesitation
- 5: Perfect recall

## Usage

```
/review             # Review all due items for active course
```

## Output

- Updated `state/progress/{course-id}/spaced-repetition.yaml`
- Review summary displayed
