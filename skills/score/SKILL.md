---
name: score
description: View scores, trends, and recommendations
triggers: [score, scores, progress, how am I doing]
---

## What This Does

Generates a scorecard with actionable recommendations:

Per module:
- Composite score (weighted: study 20%, mastery 50%, tutor 30%)
- Bloom-level breakdown (where you excel vs. struggle)
- Trend: improving | stable | declining

Cumulative:
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

## Workflow

Execute `workflows/definitions/score-review.yaml`

## Output

- Scorecard displayed in session
- `output/{course-id}/score-report.md` written
