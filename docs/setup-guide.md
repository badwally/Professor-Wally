# Setup Guide

Step-by-step walkthrough for first-run setup of the AI Tutor system.

---

## 1. Prerequisites

Before you begin, make sure you have:

- **Claude Code** installed and working (`claude` command available in your terminal)
- **Git** installed
- **Course materials** ready in markdown, PDF, text, or Jupyter notebook format
- **Optional integrations** (not required -- the system works fully without them):
  - Gmail MCP server configured in Claude Code (for Substack newsletter scanning)
  - Notion MCP server configured in Claude Code (for syncing study notes)
  - Firecrawl skill installed (`firecrawl:firecrawl-cli`) for web research enrichment

---

## 2. Clone and Launch

```bash
git clone https://github.com/badwally/Professor-Wally.git ~/ai-tutor
cd ~/ai-tutor
claude
```

This opens Claude Code in the AI Tutor project directory. The system reads `CLAUDE.md` on startup and loads the tutor protocol automatically.

---

## 3. Place Course Materials

Create a directory for your course materials organized by module:

```
materials/
  your-course-id/
    mod-01/
      study-guide.md       # Primary content (markdown preferred)
      transcript.pdf       # Optional: lecture transcripts
      quick-reference.md   # Optional: summary/cheat sheet
    mod-02/
      study-guide.md
      workbook.txt
    mod-03/
      study-guide.md
      assignment.ipynb
```

**Guidelines:**

- Use a short, descriptive course ID slug (e.g., `mit-ai-xpro`, `product-strategy`)
- Organize into one subdirectory per module: `mod-01/`, `mod-02/`, etc.
- Naming is flexible -- the ingest step normalizes variants like `Mod 1`, `Module 1`, `module-01`
- Supported file types: `.md` (preferred), `.pdf`, `.txt`, `.ipynb`
- Each module folder should have at least one primary content file (a study guide or main reading)

---

## 4. Run `/setup`

In your Claude Code session, type:

```
/setup
```

The setup wizard walks you through five steps. Here is what to expect at each stage.

### Step 1: Welcome

The system introduces the five setup steps and asks you to confirm before proceeding.

### Step 2: Learning Profile

You will be asked 13 questions across four categories. Answer conversationally -- the system adapts to your role and context.

**Learning Style (Questions 1-5)**

| Question | What it controls | Example of a good answer |
|----------|-----------------|-------------------------|
| Preferred learning mode (visual, auditory, reading, hands-on)? | How study guides present information | "Reading primarily, with visual diagrams for complex systems" |
| How do you take notes? | Note format suggestions | "Cornell method for lectures, freeform for brainstorming" |
| Ideal study session length (10-120 min)? | Session pacing and checkpoint timing | "45 minutes -- I lose focus after an hour" |
| When are you sharpest? | Scheduling recommendations | "Mornings 6-8am, and a second wind around 8-10pm" |
| Breadth-first or depth-first? | Content ordering strategy | "Depth-first -- I want to fully understand one topic before moving on" |

**Motivation (Questions 6-8)**

| Question | What it controls | Example of a good answer |
|----------|-----------------|-------------------------|
| Main learning goals? | Objective alignment and progress framing | "Pass the MIT AI certification, be able to build AI product roadmaps at work" |
| Accountability style? | How the system nudges you | "Deadline-driven -- I need hard milestones or I'll procrastinate" |
| What motivates you? | Reward and feedback patterns | "Streak tracking -- seeing consecutive days keeps me going" |

**Strengths and Gaps (Questions 9-10)**

| Question | What it controls | Example of a good answer |
|----------|-----------------|-------------------------|
| Strong areas? | Adjusts difficulty up for familiar topics | "Business strategy, product management, stakeholder communication" |
| Weak areas? | Allocates more time and scaffolding | "Math fundamentals, Python coding, statistical modeling" |

**Preferences (Questions 11-13)**

| Question | What it controls | Example of a good answer |
|----------|-----------------|-------------------------|
| Socratic questioning or direct explanations? | Tutoring interaction style | "Socratic for topics I partially know, direct for brand-new concepts" |
| Feedback style (encouraging, blunt, balanced)? | How scores and corrections are delivered | "Blunt -- tell me what I got wrong and why, skip the pleasantries" |
| Example preference (real-world, academic, mixed)? | Examples used in study guides and explanations | "Real-world -- I learn best from industry case studies" |

The system also asks about **Substack topics** (optional) -- keywords for scanning Gmail newsletters for supplementary material (e.g., "AI", "product management", "machine learning").

**What to expect:** After answering, the system generates `state/config/learner.yaml` and shows it to you for review. You can edit any values before confirming.

### Step 3: Course Objectives

The system asks about your course:

1. **Course name and provider** (e.g., "MIT xPRO -- Designing and Building AI Products")
2. **Top 3-5 learning objectives** -- what you will be able to do, how you measure it, and at what Bloom level (remembering, understanding, applying, analyzing, evaluating, creating)
3. **Materials location** -- file path to your course materials
4. **Module count and names** -- list them if known, or the system discovers them during ingest

**What to expect:** Generates `courses/{course-id}/course.yaml` with default scoring thresholds (passing: 0.70, fluency: 0.80, mastery: 0.90). You can review and edit before confirming.

### Step 4: Success Criteria

Define what "done" means for you:

1. **Score threshold** -- default is 80% (Proficient) to advance between modules
2. **Stricter criteria for specific modules** -- optionally require higher scores for critical modules
3. **External milestones** -- e.g., "Pass certification exam by June"
4. **Below-threshold policy** -- choose one:
   - Forced review before advancing (strictest)
   - Flag for review but allow advancement (flexible)
   - Strict: must achieve threshold to proceed (default)

**What to expect:** Updates the `success_criteria` section of your `course.yaml`.

### Step 5: Tool Integrations

Configure optional MCP connections:

1. **Notion** -- sync study notes and guides as tagged Notion pages
2. **Gmail (Substack scanning)** -- find supplementary articles from newsletters
3. **Firecrawl (Web Research)** -- augment study guides with current web sources

All three are optional. Answer "no" to skip any integration.

**What to expect:** Generates `state/config/tools.yaml`. The system initializes all state files (progress tracking, SRS queue, session log, memory) and presents a setup summary with next steps.

---

## 5. Configure Tools (Optional)

If you skipped tool configuration during setup or want to change settings later:

1. Copy the template:

```bash
cp state/config/tools.template.yaml state/config/tools.yaml
```

2. Edit `state/config/tools.yaml` and set `enabled: true` for each integration you have available:

```yaml
gmail:
  enabled: true       # Set to true if Gmail MCP server is configured

notion:
  enabled: true       # Set to true if Notion MCP server is configured
  database_id: "abc123"  # Your Notion database ID for study materials

firecrawl:
  enabled: true       # Set to true if firecrawl:firecrawl-cli skill is installed
```

**Integration details:**

| Integration | MCP Requirement | Used By |
|-------------|----------------|---------|
| Gmail | Gmail MCP server in Claude Code | `/enrich` scans Substack newsletters for supplementary material |
| Notion | Notion MCP server in Claude Code | `/enrich` syncs study guides; `/study` syncs session summaries |
| Firecrawl | `firecrawl:firecrawl-cli` skill installed | `/enrich` for web research; `/research` for deep dives |

---

## 6. Run `/ingest`

With materials in place and setup complete, ingest your course:

```
/ingest
```

**What it does (step by step):**

1. **Validates materials** -- reads `course.yaml` for the materials path, scans the directory, and normalizes folder names (e.g., "Mod 1" becomes `mod-01`)
2. **Extracts module structure** -- identifies file types per module (study guides, transcripts, workbooks, notebooks) and flags modules missing a primary study guide
3. **Extracts topics and terms** -- reads each study guide section-by-section, identifies key concepts, frameworks, definitions, case studies, and cross-references
4. **Generates module configs** -- creates a YAML config per module at `courses/{course-id}/modules/{nn}-{name}.yaml` with topics, learning outcomes, Bloom levels, and prerequisites
5. **Builds glossary** -- compiles all discovered terms into `courses/{course-id}/glossary.yaml` with definitions, source modules, and related terms
6. **Builds retrieval indexes** -- creates `indexes/concepts.yaml`, `indexes/glossary-index.yaml`, and `indexes/recent.yaml` for fast concept lookup
7. **Updates course config** -- adds the discovered module list to `course.yaml`

**What to expect:** A summary showing modules indexed, terms discovered, files processed by type, and any modules flagged as missing content. The process is incremental -- re-running `/ingest` skips already-indexed modules unless you pass `--force`.

**Output files created:**

```
courses/{course-id}/
  course.yaml             # Updated with module list
  glossary.yaml           # Course-level glossary
  modules/
    01-module-name.yaml   # Per-module config
    02-module-name.yaml
    ...

indexes/
  concepts.yaml           # Concept-to-module mapping
  glossary-index.yaml     # Unified term lookup
  recent.yaml             # Last ingest record
```

---

## 7. Run `/enrich 1`

Generate study materials for Module 1:

```
/enrich 1
```

**What it does:**

1. **Checks prerequisites** -- verifies source materials exist for the module. If already enriched, reports "Already enriched. Use `--force` to regenerate."
2. **Loads source materials** -- reads the study guide section-by-section, extracts a structured outline with key concepts
3. **Supplementary research** (optional, if tools enabled):
   - **Firecrawl**: searches the web for each topic -- foundational explanations, current applications, comparative analyses. Selects 3-5 best sources per topic.
   - **Gmail**: scans Substack newsletters matching course topics and learner keyword preferences. Extracts content from top results and maps articles to module topics.
4. **Generates enriched study guide** -- structured with a Big Picture opening, per-topic sections (concept summary, "why it matters", active recall prompts, examples, key terms), a "Beyond the Course" section (if web research was done), and a Self-Check section
5. **Generates study questions** (15-20) -- Bloom distribution: 30% Remember, 40% Understand, 20% Apply, 10% Analyze. Short-answer format with binary scoring.
6. **Generates mastery questions** (8-12) -- Bloom distribution: 25% Apply, 30% Analyze, 25% Evaluate, 20% Create. Written response format with rubric scoring, ascending difficulty.
7. **Seeds SRS queue** -- extracts flashcard-style items from questions and adds them to the spaced repetition queue with initial interval of 1 day
8. **Syncs to Notion** (optional, if enabled) -- creates or updates a study guide page in your configured Notion database

**Output files created:**

```
output/{course-id}/
  study-guides/
    module-01-study-guide.md        # Enriched study guide
  questions/
    module-01-study-questions.md    # 15-20 study questions
    module-01-mastery-questions.md  # 8-12 mastery questions
```

---

## 8. Run `/study 1`

Start your first interactive study session:

```
/study 1
```

**What a session looks like:**

1. **Session planning** -- the system checks your progress, SRS items due, session length preference, and last checkpoint. It presents a plan (sections to cover, estimated time) and asks you to confirm.

2. **SRS review** (if items are due) -- flashcard-style review of spaced repetition items. You rate your recall quality (0-5) for each item and the system updates intervals using the SM2 algorithm. Capped at 20 items, takes 5-10 minutes.

3. **Section-by-section study** -- for each section of the enriched study guide:
   - Section digest with key concepts
   - Active recall prompt: "What do you already know about this topic?"
   - Gap-filling based on your response
   - **Study questions** presented one at a time, scored immediately (correct/incorrect), with hints on wrong answers
   - **Mastery questions** requiring written responses, scored on a 5-dimension rubric (accuracy, depth, connections, examples, clarity)
   - Section score displayed after each section

4. **Checkpointing** -- the system tracks elapsed time against your session length preference. If you are approaching the limit, it asks whether to continue or save a checkpoint. You can resume from the checkpoint in your next session.

5. **Session summary** -- displays composite score, Bloom-level performance breakdown, identified gaps, and recommended next steps:
   - Below 0.80 (not yet fluent): recommends re-studying weak topics
   - At or above 0.80 (Proficient): module complete, suggests advancing

**Composite scoring formula:**

```
module_score = (study_questions * 0.20) + (mastery_questions * 0.50) + (tutor_quality * 0.30)
```

**Status levels:**

| Score | Status | Meaning |
|-------|--------|---------|
| < 0.70 | Below Passing | Needs significant review |
| 0.70 - 0.79 | Passing | Meets minimum threshold |
| 0.80 - 0.89 | Proficient | Strong understanding (fluency threshold) |
| >= 0.90 | Mastery | Deep, transferable knowledge |

For deeper understanding, use `/tutor 1` instead -- this activates Socratic mode with elaborative interrogation, teach-back exercises, and cross-module interleaving.

---

## 9. Typical Workflow After Setup

Once setup is complete, a typical study cycle looks like this:

```
/enrich 2          # Prepare the next module
/study 2           # Study it interactively
/score all         # Check overall progress
/review            # Quick SRS flashcard review
/tutor 2           # Deep dive on weak areas
/research "topic"  # Web research on a specific concept
```

---

## 10. Common Issues and Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| "Module has no study guide" | Materials not placed in the correct directory, or `/ingest` has not been run | Place files in `materials/{course-id}/mod-{nn}/` and run `/ingest` |
| "Already enriched" | Study guide output already exists for this module | Use `/enrich {n} --force` to regenerate |
| SRS items not showing up | Items exist but are not due for review yet | Check `state/progress/{course-id}/spaced-repetition.yaml` -- items have a `next_review` date |
| Low scores on mastery questions | Gaps in understanding or insufficient depth in written responses | Use `/tutor {module}` for Socratic deep dives on weak topics |
| MCP tools not working | Integration not enabled or MCP server not running | Verify `state/config/tools.yaml` has `enabled: true` and the MCP server is configured in Claude Code |
| Session interrupted | Closed Claude Code mid-session | Re-run `/study {module}` -- the system detects the checkpoint and offers to resume |
| Cannot advance to next module | Composite score below 0.80 fluency threshold | Re-study weak sections, use `/tutor` for targeted practice, then re-attempt |
| Ingest skips modules | Incremental mode skips already-indexed content | Run `/ingest --force` to re-index everything |

---

## File Reference

Key files created and used by the system:

| File | Created By | Purpose |
|------|-----------|---------|
| `state/config/learner.yaml` | `/setup` | Your learning profile and preferences |
| `state/config/tools.yaml` | `/setup` | MCP integration settings |
| `courses/{id}/course.yaml` | `/setup` | Course configuration and objectives |
| `courses/{id}/modules/*.yaml` | `/ingest` | Per-module topic configs |
| `courses/{id}/glossary.yaml` | `/ingest` | Course-level term glossary |
| `output/{id}/study-guides/*.md` | `/enrich` | Enriched study guides |
| `output/{id}/questions/*.md` | `/enrich` | Study and mastery question banks |
| `state/progress/{id}/progress.yaml` | `/setup`, `/study` | Scores and module status |
| `state/progress/{id}/spaced-repetition.yaml` | `/enrich`, `/study` | SRS flashcard queue |
| `state/progress/{id}/session-log.yaml` | `/study` | Session history and checkpoints |
| `state/memory/memory.yaml` | `/study` | Learner insights and patterns |
