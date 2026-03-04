---
name: tutor
description: Socratic tutoring session — deep understanding through questioning
user_invocable: true
---

# /tutor — Socratic Tutoring Session

Execute the study-session workflow in Socratic mode via `workflows/definitions/study-session.yaml`.

## Instructions

1. Read the workflow definition from `workflows/definitions/study-session.yaml`
2. Read policies listed in the workflow's `context.policies` from `policies/`
3. Load the active course from `state/courses/_active.yaml`
4. Parse arguments: module number, optional `--topic` flag
5. Execute in **Socratic mode** — the learner explains, you guide with questions

## What This Does

Unlike `/study` (which balances content with questions), `/tutor` puts the learner in the driver's seat. They explain, generate examples, and make connections. The tutor guides with questions, not answers.

### Techniques (rotate automatically, cycle through all 5 before repeating):

1. **Elaborative Interrogation** — "Why does this work?" / "What would happen if...?"
2. **Teach-Back (Feynman Technique)** — "Explain this as if teaching a colleague."
3. **Concrete Example Generation** — "Give me a real-world example of this."
4. **Cross-Module Interleaving** — "How does this connect to what you learned in Module N?"
5. **Socratic Questioning Chain** — Progressive narrowing from broad to specific.

### Scoring (5 dimensions, each 0-100):

| Dimension | What It Measures |
|-----------|-----------------|
| Recall | Can you retrieve relevant facts? |
| Understanding | Do you explain mechanisms, not just terms? |
| Application | Can you use the concept in a new context? |
| Integration | Do you connect to other concepts/modules? |
| Teaching | Can you explain it clearly to others? |

Composite = average of all 5 dimensions / 100 (0.00-1.00)

### Meta-Feedback after each exchange:
- **Strong**: Reasoning identified mechanisms and anticipated counter-arguments
- **Developing**: Identified the right concept but reasoning needs more depth
- **Emerging**: Start with the mechanism (why) before the outcome (what)

## Usage

```
/tutor 2                         # Tutor session on Module 2
/tutor 1 --topic "Delta Model"   # Focus on a specific topic
```

## Prerequisites

- Module must be enriched (`/enrich <module>`)

## Output

- Per-interaction scores across 5 dimensions
- Session composite score
- Updated progress, SRS items, session log, and memory entries
