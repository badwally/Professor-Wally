---
id: spaced-repetition
scope: always-on
---

## Rule

Implement spaced repetition using a modified SM2 (SuperMemo 2) algorithm for all SRS items.

### SM2 Algorithm

After each review, the learner self-rates recall quality on a 0-5 scale:
- **0**: Complete blackout — no memory at all
- **1**: Incorrect — but recognized the answer when shown
- **2**: Incorrect — but the answer felt familiar
- **3**: Correct — but required significant effort to recall
- **4**: Correct — with some hesitation
- **5**: Perfect — instant recall with confidence

### Interval Calculation

```
If quality < 3:
    repetitions = 0
    interval = 1 day
    (Item is "forgotten" — restart from scratch)

If quality >= 3:
    If repetitions == 0: interval = 1 day
    If repetitions == 1: interval = 6 days
    If repetitions >= 2: interval = previous_interval * ease_factor

    ease_factor = max(1.3, ease_factor + (0.1 - (5 - quality) * (0.08 + (5 - quality) * 0.02)))
    repetitions += 1
```

### Default Values for New Items
- `interval_days`: 1
- `ease_factor`: 2.5
- `repetitions`: 0
- `next_review`: today's date

### SRS Item Types
- **concept**: Key term or definition (Bloom: remember)
- **application**: Scenario-based question (Bloom: apply/analyze)
- **connection**: Cross-module relationship (Bloom: evaluate)

### Session Integration
1. At session start, identify all items where `next_review <= today`
2. Present due items before new material
3. After each item: show answer, ask for self-rating (0-5)
4. Update the item's interval, ease_factor, repetitions, and next_review
5. Append to the item's history array

### Item Generation
- Study questions automatically seed SRS items (front=question, back=answer)
- Mastery questions seed items for the key concepts they test (not the full question)
- The tutor can manually add items during sessions for concepts that need reinforcement
- Aim for 5-10 new SRS items per module

### Retention Target
- Target: 85%+ of items at quality >= 3 on most recent review
- Track retention rate in progress.yaml as `srs_retention_rate`
- If retention drops below 70%, recommend dedicated review sessions

## Exceptions

- If the SRS queue has > 30 items due, cap the review at 20 items per session
- Allow the learner to skip SRS review with explicit acknowledgment ("I want to skip review today")
