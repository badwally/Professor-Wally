---
name: quiz
description: Scored assessment — evaluative only, not instructional
user_invocable: true
---

# /quiz — Scored Assessment

Execute the quiz workflow defined in `workflows/definitions/quiz.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/quiz.yaml`
2. Read policies listed in the workflow's `context.policies` from `policies/`
3. Load the active course from `state/courses/_active.yaml`
4. Parse arguments: module number, `cumulative`, or `--focus` topic
5. Execute the workflow steps in order, respecting all gates

## What This Does

A focused quiz that evaluates knowledge without teaching:
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

## Prerequisites

- Module must be enriched (questions must exist)

## Output

- Quiz scores displayed with breakdown
- Updated `state/progress/{course-id}/progress.yaml`
- Updated `state/progress/{course-id}/spaced-repetition.yaml`
