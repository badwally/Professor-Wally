# AI Tutor

A Claude Code-based study system that ingests course materials, generates enriched study guides, and tests understanding with progressive questions. Tracks learning progress across modules with spaced repetition and evidence-based pedagogy.

## Quick Start

```bash
# 1. Open ai-tutor in Claude Code
cd ~/ai-tutor
claude

# 2. Run first-time setup
/setup

# 3. Place course materials in materials/{course-id}/
# 4. Ingest and index the course
/ingest

# 5. Enrich the first module (generates study guide + questions)
/enrich 1

# 6. Start studying
/study 1
```

## Commands

| Command | Purpose |
|---------|---------|
| `/setup` | First-run: create learner profile, configure tools |
| `/ingest` | Scan and index course materials |
| `/enrich <module>` | Generate enriched study guide + questions |
| `/study [module]` | Interactive guided study session |
| `/quiz [module\|cumulative]` | Scored assessment |
| `/tutor [module]` | Socratic tutoring with teach-back |
| `/score [module\|all]` | View scores and recommendations |
| `/review` | Quick spaced repetition review |
| `/research <topic>` | Web research deep dive |
| `/switch <course-id>` | Switch active course |

## Study Session Flow

1. New module materials placed in course folder
2. `/enrich <module>` generates study guide + questions (.md output)
3. `/study <module>` walks you through:
   - Read section of enriched study guide
   - Answer study questions (factual recall)
   - Answer mastery questions (written, progressive difficulty)
   - See section score and feedback
4. At session end: module score + gaps identified
5. Next session: re-study weak areas, continue module, or advance

## Scoring

- **Study Questions** (20%): Factual recall, correct/incorrect
- **Mastery Questions** (50%): Written analysis, rubric-scored
- **Tutor Sessions** (30%): 5-dimension quality assessment

Fluency threshold to advance: **0.80 (Proficient)**

## Architecture

Adapted from the [chief-of-staff](https://github.com/badwally/chief-of-staff) 4-layer pattern:

```
L3: Interface    — Skills (/study, /quiz, /tutor, etc.)
L2: Workflows    — Declarative YAML (enrich, study-session, quiz, etc.)
L1: Retrieval    — Indexes (concepts, glossary, recent)
L0: State        — YAML/Markdown (progress, SRS, memory, configs)
```

All state stored as human-readable YAML/Markdown files. No external databases.

## Adding a New Course

1. Create `courses/{new-id}/course.yaml` (use template)
2. Place materials in `materials/{new-id}/`
3. Run `/ingest` to auto-generate module configs and glossary
4. Run `/switch {new-id}` to activate
5. Run `/enrich 1` to generate first study guide

See `docs/course-authoring.md` for details.

## MCP Integrations (Optional)

- **Gmail**: Scan Substack newsletters for supplementary material
- **Notion**: Sync tagged study notes and guides
- **Firecrawl**: Web research to enrich guides with current sources

All integrations are optional. The system works fully with just course materials.
