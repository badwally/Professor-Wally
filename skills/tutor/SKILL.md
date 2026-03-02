---
name: tutor
description: Socratic tutoring session — deep understanding through questioning
triggers: [tutor, teach me, explain, help me understand, tutor session]
---

## What This Does

Runs an interactive Socratic tutoring session that prioritizes deep understanding over information delivery. Unlike `/study` (which balances content presentation with questions), `/tutor` puts YOU in the driver's seat — you explain, you generate examples, you make connections. The tutor guides with questions, not answers.

## Techniques Used

1. **Elaborative Interrogation** — "Why does this work?" / "What would happen if...?"
   Forces you to articulate the mechanism behind concepts, not just the outcome.

2. **Teach-Back (Feynman Technique)** — "Explain this as if teaching a colleague."
   Tests whether you truly understand or are just pattern-matching terminology.

3. **Concrete Example Generation** — "Give me a real-world example of this."
   Validates that you can transfer abstract concepts to concrete situations.

4. **Cross-Module Interleaving** — "How does this connect to what you learned in Module N?"
   Builds transfer ability by linking concepts across module boundaries.

5. **Socratic Questioning Chain** — Progressive narrowing from broad to specific.
   Starts with what you know, then drills into mechanisms, alternatives, and evidence.

Techniques rotate automatically — the tutor cycles through all 5 before repeating any.

## Scoring

Tutor sessions are scored on 5 dimensions (each 0-100):

| Dimension | What It Measures |
|-----------|-----------------|
| **Recall** | Can you retrieve relevant facts? |
| **Understanding** | Do you explain mechanisms, not just terms? |
| **Application** | Can you use the concept in a new context? |
| **Integration** | Do you connect to other concepts/modules? |
| **Teaching** | Can you explain it clearly to others? |

**Composite** = average of all 5 dimensions / 100 (0.00-1.00)

The composite feeds into the module score: `(study * 0.20) + (mastery * 0.50) + (tutor * 0.30)`

## Meta-Feedback

After each exchange, you'll receive reasoning-quality feedback:
- **Strong**: Your reasoning identified mechanisms and anticipated counter-arguments
- **Developing**: You identified the right concept but reasoning needs more depth
- **Emerging**: Start with the mechanism (why) before the outcome (what)

This feedback is tracked across sessions to identify reasoning patterns and effective techniques.

## Usage

```
/tutor 2                         # Tutor session on Module 2
/tutor 1 --topic "Delta Model"   # Focus on a specific topic within Module 1
/tutor 3 --topic "LLM Agents"    # Deep dive on a single topic
```

### Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--topic` | Focus on a specific topic within the module | All topics |

## Prerequisites

- Module must be enriched (`/enrich <module>` first)
- Study guide and questions must exist in `output/{course-id}/`

## Example Session Flow

```
You:   /tutor 2 --topic "Delta Model"
Tutor: What do you already know about the Delta Model?
You:   It's about how AI changes product strategy...
Tutor: Good start. WHY does AI change product strategy specifically?
       What's the mechanism that makes traditional approaches insufficient?
You:   Because AI products have different feedback loops...
Tutor: Interesting — explain what you mean by "different feedback loops."
       How do they differ from traditional software feedback loops?
You:   [explains]
Tutor: Strong reasoning — you identified the continuous learning mechanism.
       Now explain the Delta Model as if teaching a colleague who only
       knows traditional product management.
You:   [teach-back attempt]
Tutor: [scores + meta-feedback + next technique]
```

## Workflow

Executes `workflows/definitions/study-session.yaml` in Socratic mode.

## Output

- Per-interaction scores across 5 dimensions
- Session composite score
- Meta-feedback on reasoning patterns
- Updated progress, SRS items, and session log
- Memory entries for effective techniques and reasoning patterns
