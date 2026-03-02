---
name: quiz
description: Scored assessment — evaluative only, not instructional
triggers: [quiz, test me, test, assessment]
---

## What This Does

A focused quiz that evaluates your knowledge without teaching:
1. Selects questions based on scope (module or cumulative)
2. Prioritizes: low-score topics, SRS-due items, interleaved modules
3. Presents questions one at a time with immediate feedback
4. Scores per-question, per-Bloom-level, per-topic, and overall
5. Updates progress and SRS

## Usage

```
/quiz 2             # Quiz on Module 2
/quiz cumulative    # Quiz across all completed modules
/quiz 1 --focus "Delta Model"  # Focus on specific topic
```

## Workflow

Execute `workflows/definitions/quiz.yaml`

## Prerequisites

- Module must be enriched (questions must exist)

## Output

- Quiz scores displayed with breakdown
- Updated `state/progress/{course-id}/progress.yaml`
- Updated `state/progress/{course-id}/spaced-repetition.yaml`
