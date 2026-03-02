---
name: tutor
description: Socratic tutoring session — deep understanding through questioning
triggers: [tutor, teach me, explain, help me understand, tutor session]
---

## What This Does

Like `/study` but with heavier emphasis on deep understanding:
- **Elaborative interrogation**: "Why does this work?" at every concept
- **Teach-back / Feynman technique**: You explain concepts as if teaching
- **Concrete example generation**: You create examples, tutor validates
- **Cross-module interleaving**: Connects current material to prior modules
- **Socratic questioning**: Guides you to answers through questions, not explanations

## Usage

```
/tutor 2                # Tutor session on Module 2
/tutor 1 --topic "Delta Model"  # Focus on a specific topic
```

## Workflow

Execute `workflows/definitions/study-session.yaml` in Socratic mode

## Prerequisites

- Module must be enriched

## Output

- Same as `/study` but with emphasis on tutor session quality scores
