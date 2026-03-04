---
name: score
description: View scores, trends, and recommendations
user_invocable: true
---

# /score — View Scores and Recommendations

Execute the score-review workflow defined in `workflows/definitions/score-review.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/score-review.yaml`
2. Load the active course from `state/courses/_active.yaml`
3. Load `state/progress/{course-id}/progress.yaml`
4. Parse arguments: module number, `all`, or no argument (full scorecard)
5. Execute the workflow steps in order

## What This Does

Generates a scorecard with actionable recommendations:

**Per module:**
- Composite score (weighted: study 20%, mastery 50%, tutor 30%)
- Bloom-level breakdown (where you excel vs. struggle)
- Trend: improving | stable | declining

**Cumulative:**
- Overall course score
- Strongest/weakest topics
- SRS retention rate
- Total study hours and streak
- Recommendations for what to do next

## Usage

```
/score              # Full scorecard for active course
/score 2            # Score for Module 2 only
/score all          # All courses
```

## Output

- Scorecard displayed in session
- `output/{course-id}/score-report.md` written
