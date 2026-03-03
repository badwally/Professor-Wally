# AI Tutor -- Architecture Reference

## 1. Overview

The AI Tutor is a Claude Code-based system that guides learners through course material using evidence-based study techniques: spaced repetition, active recall, teach-back, interleaving, and elaborative interrogation. It never presents information passively -- it always engages the learner first.

The system is adapted from the [chief-of-staff](https://github.com/coleam00/chief-of-staff) pattern and uses three core architectural principles:

- **Declarative YAML workflows.** Every user-facing operation (`/study`, `/quiz`, `/enrich`, etc.) is defined as a sequence of steps in a YAML file. Claude Code reads the definition and executes it.
- **Filesystem-as-database.** All learner state, progress, scores, and memory live in YAML files on disk. No external database. Git tracks templates and configs; personal state files are gitignored.
- **Always-on policies.** Markdown files in `policies/` are loaded at session start and constrain every interaction -- pedagogical approach, scoring rubric, SRS rules, context discipline, anti-pattern guards.

## 2. Four-Layer Architecture

```
+------------------------------------------------------------------+
|  L3  Interface         skills/               /command routing     |
+------------------------------------------------------------------+
|  L2  Workflows         workflows/definitions/   step execution   |
+------------------------------------------------------------------+
|  L1  Retrieval         indexes/             concept & term lookup |
+------------------------------------------------------------------+
|  L0  State             state/               learner data on disk  |
+------------------------------------------------------------------+
```

### L0 -- State (`state/`)

The persistence layer. All learner data lives here as YAML files.

| Path | Purpose |
|------|---------|
| `state/config/learner.yaml` | Learner profile -- learning style, preferences, goals |
| `state/config/tools.yaml` | MCP integration toggles (Gmail, Notion, Firecrawl) |
| `state/config/modes.yaml` | Keyword triggers for mode inference and skill routing |
| `state/courses/_active.yaml` | Pointer to the currently active course |
| `state/progress/{course-id}/progress.yaml` | Per-module scores, status, gaps, Bloom breakdown |
| `state/progress/{course-id}/spaced-repetition.yaml` | SM2 SRS queue -- items, intervals, ease factors |
| `state/progress/{course-id}/session-log.yaml` | Append-only session history with checkpoints |
| `state/memory/memory.yaml` | Learner insights -- patterns, effective techniques, growth areas |

Templates (tracked in git): `state/config/learner.template.yaml`, `state/config/tools.template.yaml`. Personal files (gitignored): everything generated from those templates.

Schemas defining the structure of state files live in `workflows/schemas/`:
- `learner-profile.yaml` -- learner config validation
- `course-config.yaml` -- course structure
- `module-config.yaml` -- per-module configuration
- `question-set.yaml` -- study and mastery question format
- `quiz-result.yaml` -- quiz outcome structure
- `score-report.yaml` -- scorecard format
- `tutor-session.yaml` -- tutor interaction scoring

### L1 -- Retrieval (`indexes/`)

Lookup layer built during `/ingest` and updated by workflows. Enables targeted reads instead of scanning full materials.

| Index File | Contents |
|------------|----------|
| `indexes/concepts.yaml` | Maps concept names to modules and source files (type: framework, model, technique, etc.) |
| `indexes/glossary-index.yaml` | Unified term lookup across all courses with module sources and related terms |
| `indexes/recent.yaml` | Tracks last ingest timestamp and which modules were indexed |

Built by the `build-indexes` step of `/ingest`. Referenced by `/study`, `/enrich`, `/quiz`, and `/research` workflows via the `context.indexes` field.

### L2 -- Workflows (`workflows/definitions/`)

The execution layer. Each workflow is a declarative YAML file with:

- **Steps** -- Sequential execution. Each step has a `name`, `executor` (always `prompt` in this system), a `prompt` template, `input` bindings, and `output` declarations.
- **Input chaining** -- Steps reference prior outputs via `${step-name.key}` syntax. Example: `${assess-state.session_plan}` passes the plan from step 1 to step 2.
- **Gates** -- Pause points requiring user interaction before proceeding:
  - `confirm` -- user must approve before continuing
  - `display` -- show output to user (terminal gate, usually final step)
  - `edit` -- user can modify the generated output before it is saved
  - `approve` -- user must explicitly accept
- **Optional steps** -- Steps marked `optional: true` are skipped when their MCP integration is unavailable. The system checks `state/config/tools.yaml` at runtime.
- **Context** -- Each workflow declares which `policies`, `memory_scopes`, and `indexes` it needs loaded.

Workflow definitions:

| File | Trigger | Steps |
|------|---------|-------|
| `setup.yaml` | `/setup` | welcome, learning style, course objectives, success criteria, tools, init state, summary |
| `ingest-course.yaml` | `/ingest` | validate materials, extract structure, extract topics/terms, generate configs, generate glossary, build indexes, update course config, summary |
| `enrich-module.yaml` | `/enrich` | check prerequisites, load sources, supplementary research (opt), generate study guide, generate questions, seed SRS, sync to Notion (opt), summary |
| `study-session.yaml` | `/study`, `/tutor` | assess state, SRS review (opt), section study loop, compute scores, update state, session summary, sync to Notion (opt) |
| `quiz.yaml` | `/quiz` | configure quiz, administer, score, update progress, summary |
| `score-review.yaml` | `/score` | load progress, compute scorecard, generate report, display |
| `research-augment.yaml` | `/research` | define research, execute search, synthesize, sync and report (opt) |

### L3 -- Interface (`skills/`)

The routing layer. Claude Code skills map `/command` invocations and keyword triggers to workflow definitions.

Routing is configured in `state/config/modes.yaml`, which defines:
- **Modes** -- Named operating contexts (study, tutor, assess, research, enrich) with keyword triggers and required policies.
- **Skills** -- Each skill points to a `SKILL.md` file (the Claude Code skill definition) and a workflow YAML file. The skill file tells Claude Code how to invoke the workflow.

Keyword matching allows natural language triggers. For example, "help me understand module 3" matches the `tutor` skill triggers (`["tutor", "teach me", "explain", "help me understand"]`) and routes to `study-session.yaml` in Socratic mode.

Some skills are inline (no workflow file): `/review` runs SRS review directly, `/switch` updates `_active.yaml`.

## 3. Data Flow Diagrams

### `/setup` -- First-Run Configuration

```
User runs /setup
       |
       v
  +-----------+     +-----------------+     +-----------------+
  |  welcome  |---->| learning style  |---->| course          |
  |  (confirm)|     | questions       |     | objectives      |
  +-----------+     | (gate: edit)    |     | (gate: edit)    |
                    +--------+--------+     +--------+--------+
                             |                       |
                    +--------v--------+     +--------v--------+
                    | learner.yaml    |     | course.yaml     |
                    +-----------------+     +-----------------+
                                                     |
                    +-----------------+     +--------v--------+
                    | tools.yaml      |<----| success         |
                    | (gate: edit)    |     | criteria        |
                    +--------+--------+     +-----------------+
                             |
                    +--------v--------+
                    | initialize      |
                    | state files     |
                    +--------+--------+
                             |
                             v
              _active.yaml, progress.yaml,
              spaced-repetition.yaml,
              session-log.yaml, memory.yaml
```

### `/ingest` -- Course Material Indexing

```
User runs /ingest
       |
       v
  +------------------+     +------------------+     +------------------+
  | validate         |---->| extract module   |---->| extract topics   |
  | materials dir    |     | structure        |     | and terms        |
  +------------------+     +------------------+     +--------+---------+
                                                             |
                      +--------------------------------------+
                      |                    |                  |
             +--------v-------+  +--------v-------+  +------v--------+
             | generate       |  | generate       |  | build         |
             | module configs |  | glossary.yaml  |  | indexes       |
             +--------+-------+  +----------------+  +------+--------+
                      |                                      |
                      v                                      v
            courses/{id}/modules/                  indexes/concepts.yaml
            {nn}-{name}.yaml                       indexes/glossary-index.yaml
                                                   indexes/recent.yaml
```

### `/enrich` -- Study Guide Generation

```
User runs /enrich <module>
       |
       v
  +------------------+     +------------------+
  | check            |---->| load source      |
  | prerequisites    |     | materials        |
  +------------------+     +--------+---------+
                                    |
                      +-------------+-------------+
                      |                           |
             +--------v--------+         +--------v--------+
             | Gmail: scan     |         | Firecrawl: web  |
             | Substack        |         | research        |
             | (optional)      |         | (optional)      |
             +--------+--------+         +--------+--------+
                      |                           |
                      +-------------+-------------+
                                    |
                           +--------v--------+
                           | generate study  |
                           | guide (t=0.6)   |
                           +--------+--------+
                                    |
                           +--------v--------+
                           | generate        |
                           | questions       |
                           | (t=0.5)         |
                           +--------+--------+
                                    |
                      +-------------+-------------+
                      |                           |
             +--------v--------+         +--------v--------+
             | seed SRS queue  |         | sync to Notion  |
             |                 |         | (optional)      |
             +--------+--------+         +-----------------+
                      |
                      v
             output/{id}/study-guides/
             output/{id}/questions/
             state/progress/{id}/spaced-repetition.yaml
```

### `/study` -- Interactive Study Session

```
User runs /study [module] or /tutor [module]
       |
       v
  +------------------+
  | assess state     |
  | (load progress,  |
  |  SRS, checkpoint,|
  |  memory, prefs)  |
  | (gate: confirm)  |
  +--------+---------+
           |
  +--------v---------+
  | SRS review       |  <-- only if items due today
  | (SM2 algorithm,  |      cap at 20 items
  |  self-rated 0-5) |
  +--------+---------+
           |
  +--------v---------+
  | section study    |  STANDARD: digest -> recall -> questions -> scoring
  | loop             |  SOCRATIC: + elaborative interrogation, teach-back,
  | (per section,    |            concrete examples, interleaving
  |  with checkpoint |  Checkpoint saved after each section
  |  after each)     |  Pacing checks against session_length preference
  +--------+---------+
           |
  +--------v---------+     +------------------+     +------------------+
  | compute scores   |---->| update state     |---->| session summary  |
  | (composite,      |     | (progress, SRS,  |     | (gate: display)  |
  |  Bloom, trend)   |     |  session log,    |     +--------+---------+
  +------------------+     |  memory)         |              |
                           +------------------+     +--------v--------+
                                                    | sync to Notion  |
                                                    | (optional)      |
                                                    +-----------------+
```

### `/quiz` -- Standalone Assessment

```
User runs /quiz [module|cumulative]
       |
       v
  +------------------+     +------------------+     +------------------+
  | configure quiz   |---->| administer quiz  |---->| score quiz       |
  | (select scope,   |     | (one question at |     | (per-Bloom,      |
  |  question mix,   |     |  a time, score   |     |  per-topic,      |
  |  interleaving)   |     |  immediately)    |     |  per-module,     |
  | (gate: confirm)  |     +------------------+     |  trend)          |
  +------------------+                              +--------+---------+
                                                             |
                                                    +--------v--------+
                                                    | update progress |
                                                    | (keep higher    |
                                                    |  score, seed    |
                                                    |  SRS for wrong) |
                                                    +--------+--------+
                                                             |
                                                    +--------v--------+
                                                    | quiz summary    |
                                                    | (gate: display) |
                                                    +-----------------+
```

### `/score` -- Score Review

```
User runs /score [module|all]
       |
       v
  +------------------+     +------------------+     +------------------+
  | load all         |---->| compute          |---->| generate report  |
  | progress data    |     | scorecard        |     | (markdown)       |
  | (progress, SRS,  |     | (composite,      |     +--------+---------+
  |  sessions, prefs)|     |  Bloom, trend,   |              |
  +------------------+     |  recommendations)|     +--------v--------+
                           +------------------+     | display          |
                                                    | scorecard        |
                                                    | (gate: display)  |
                                                    +-----------------+
                                                             |
                                                             v
                                                    output/{id}/score-report.md
```

## 4. State Management

State files form an interconnected web. Each file has specific workflows that read from and write to it.

```
                    +-------------------+
                    |   _active.yaml    |<------------ /switch (write)
                    | (current course)  |<------------ session-start (read)
                    +-------------------+

+-------------------+                         +------------------------+
|  learner.yaml     |<--- /setup (write)      |  tools.yaml            |
| (learning style,  |<--- all workflows       | (MCP toggles)          |
|  preferences)     |     (read)              | gmail, notion,         |
+-------------------+                         | firecrawl enabled?     |
                                              +------------------------+
                                                  ^
                                                  | read by optional steps

+-------------------+     +-------------------+     +-------------------+
|  progress.yaml    |     | spaced-            |     |  session-log.yaml |
| (module scores,   |     | repetition.yaml   |     | (session history, |
|  status, gaps,    |     | (SRS items,        |     |  checkpoints)     |
|  Bloom breakdown) |     |  intervals, ease)  |     +-------------------+
+-------------------+     +-------------------+       ^           ^
  ^     ^     ^             ^     ^     ^     ^        |           |
  |     |     |             |     |     |     |        |           |
  |     |     +-- /score    |     |     |     |        |           |
  |     |        (read)     |     |     |     |        |           |
  |     |                   |     |     |     |        |           |
  |     +-- /quiz           |     |     |     +--/study, /quiz     |
  |        (read/write)     |     |     |       (append)           |
  |                         |     |     |                          |
  +-- /study                |     |     +-- /review                |
     (read/write)           |     |        (read/write)            |
                            |     |                                |
                            |     +-- /study                       +-- /study
                            |        (read/write,                     (append,
                            |         seed new items)                  checkpoint)
                            |
                            +-- /enrich
                               (seed new items)

+-------------------+
|  memory.yaml      |<--- /study, /tutor (write -- learner insights)
| (learning         |<--- session-start (read -- inform tutoring)
|  insights,        |
|  patterns,        |     Insights require 2+ observations before persisting.
|  effective        |     Proposed at session end, written after user approval.
|  techniques)      |
+-------------------+
```

### State file interaction summary

| State File | Read By | Written By |
|------------|---------|------------|
| `_active.yaml` | Session start, all workflows | `/switch`, `/setup` |
| `learner.yaml` | All workflows (preferences) | `/setup` |
| `tools.yaml` | Optional workflow steps | `/setup` |
| `progress.yaml` | `/study`, `/quiz`, `/score`, session start | `/study`, `/quiz` |
| `spaced-repetition.yaml` | `/study`, `/review`, session start | `/enrich` (seed), `/study` (update), `/quiz` (seed incorrect) |
| `session-log.yaml` | `/study` (checkpoint resume), session start | `/study`, `/quiz` (append) |
| `memory.yaml` | `/study`, `/tutor`, session start | `/study`, `/tutor` (session end, user-approved) |

## 5. Scoring System

### Composite Formula

```
module_score = (study_questions * 0.20) + (mastery_questions * 0.50) + (tutor_sessions * 0.30)
```

The three components weight assessment depth: mastery questions count most because they test higher-order thinking; tutor sessions capture reasoning quality; study questions verify factual recall.

### Component Scoring

**Study Questions (weight: 0.20)**
- Binary scoring: 0 (incorrect) or 1 (correct)
- No partial credit -- factual recall is either right or wrong
- Score = correct / total
- Bloom distribution: Remember 30%, Understand 40%, Apply 20%, Analyze 10%

**Mastery Questions (weight: 0.50)**
- 5-dimension rubric, each scored on a 6-point scale (0.0, 0.2, 0.4, 0.6, 0.8, 1.0):

| Dimension | What It Measures |
|-----------|-----------------|
| Accuracy | Are facts and concepts correct? |
| Depth | Beyond surface-level, explains mechanisms |
| Connections | Links to related concepts |
| Examples | Relevant, original examples |
| Clarity | Logical structure, clear expression |

- Score = average of all five dimensions (already normalized to 0-1)
- Bloom distribution: Apply 25%, Analyze 30%, Evaluate 25%, Create 20%

**Tutor Sessions (weight: 0.30)**
- 5-dimension scoring, each 0-100:

| Dimension | What It Measures |
|-----------|-----------------|
| Recall | Can they retrieve relevant facts? |
| Understanding | Do they explain mechanisms, not just terms? |
| Application | Can they use the concept in a new context? |
| Integration | Do they connect to other concepts/modules? |
| Teaching | Can they explain it clearly to others? |

- Score = average of five dimensions / 100 (normalized to 0-1)
- Only scored in Socratic mode (`/tutor` or `/study` with socratic flag)
- Each interaction also receives meta-feedback: quality rating (strong / developing / emerging), rationale, and technique suggestion

### Status Levels

| Status | Score Range | Meaning |
|--------|------------|---------|
| Below Passing | < 0.70 | Needs significant review |
| Passing | 0.70 - 0.79 | Meets minimum threshold |
| Proficient | 0.80 - 0.89 | Strong understanding |
| Mastery | >= 0.90 | Deep, transferable knowledge |

**Fluency threshold: 0.80 (Proficient).** A module must reach Proficient status before the system recommends advancing to the next module.

### Bloom Breakdown

Scores are tracked per Bloom level (Remember, Understand, Apply, Analyze, Evaluate, Create) independently of the composite score. This reveals where a learner is strong at recall but weak at application, or vice versa.

### Trend Calculation

- Window: last 3 sessions for a given module
- Improving: score increased >= 5% across the window
- Stable: change < 5% in either direction
- Declining: score decreased >= 5% across the window

### Score Report Structure

The `/score` workflow outputs a markdown report following the `score-report.yaml` schema:
- **Header**: course name, generated timestamp, total study hours, streak days
- **Per-module**: composite score, component scores, Bloom breakdown, trend, strongest/weakest topics, gaps, session count, last session date
- **Cumulative**: modules completed/total, overall score, SRS retention rate, SRS items active, ordered recommendations

Written to `output/{course-id}/score-report.md`.

## 6. Patterns from Chief-of-Staff

The system adapts the chief-of-staff pattern -- an AI executive assistant built on Claude Code -- into a learning context. Key pattern mappings:

| Chief-of-Staff Pattern | AI Tutor Adaptation |
|------------------------|---------------------|
| 4-layer architecture (State, Retrieval, Workflows, Interface) | Same four layers, repurposed: state tracks learning progress instead of task management |
| Declarative YAML workflows with steps, gates, input chaining | Identical pattern used for all 7 workflow definitions |
| Setup wizard creating personal YAML config | `/setup` creates `learner.yaml` from `learner.template.yaml` via conversational wizard |
| Filesystem-as-database (YAML/MD on disk) | All state in YAML files; git tracks templates, gitignore protects personal data |
| Always-on policies loaded as markdown | 5 pedagogical policies replace governance policies |
| Working memory with decay | `memory.yaml` captures learner insights; requires 2+ observations to persist (decay of one-off signals) |
| Graceful MCP degradation (optional steps) | Same pattern: Gmail, Notion, Firecrawl steps marked `optional: true`, skip silently when disabled |
| Session start/end protocols in CLAUDE.md | Adapted for learning: load progress, surface SRS due items, report streak, check for checkpoint |
| Skills triggered by `/command` or keyword matching | Same mechanism via `modes.yaml` triggers and skill routing |

## 7. Patterns from Open Source Research

Two open-source projects contributed specific patterns to the design.

### DeepTutor

An AI tutoring system focused on deep understanding through adaptive question generation.

| Pattern | How It Was Adapted |
|---------|--------------------|
| Incremental ingestion | `/ingest` uses `incremental: true` -- skips already-indexed modules unless `--force` flag is passed |
| Dynamic topic discovery | `/enrich` supplementary research step flags "Discovered Topics" found during web research that are not in the module config |
| Session checkpoint/resumption | `session-log.yaml` stores checkpoint state (module, section index, timestamp, techniques used); session start checks for interrupted sessions and offers to resume |
| Structured content extraction | `/ingest` extracts topics, subtopics, key terms, frameworks, definitions, and case studies using LLM-driven identification (not just keyword matching) |
| Single-pass question validation | `/enrich` generates and validates questions in one pass -- no generate-then-review-then-regenerate loops |

### Multi-Agent Study Assistant

A multi-agent system that coordinates tutoring, assessment, and content generation agents.

| Pattern | How It Was Adapted |
|---------|--------------------|
| Temperature calibration per workflow step | Each workflow step uses calibrated temperature: 0.3 for scoring, 0.5 for questions, 0.6 for study guides, 0.7 for tutoring and research |
| Learning style as prompt primitive | `learner.yaml` preferences (feedback style, example preferences, session length) are loaded as input to prompt templates, shaping all generated content |
| YAML-driven prompt configuration | Reusable prompt templates in `prompts/` (tutoring, scoring, question generation, study guide generation, enrichment, notion sync, gmail scan) are referenced by workflow steps via `reads` |

## 8. MCP Integration Points

All MCP integrations are optional. The system functions fully without any of them. Availability is controlled by `state/config/tools.yaml`.

| Integration | Used By | Purpose | Graceful Degradation |
|-------------|---------|---------|---------------------|
| **Gmail** | `/enrich` (supplementary-research step) | Scan Substack newsletters for articles matching course topics; extract content for study guide enrichment | Step skipped silently; study guide generated from course materials only |
| **Notion** | `/enrich` (sync-to-notion step), `/study` (sync-to-notion step), `/research` (sync-and-report step) | Sync study guides, session summaries, and research notes to a Notion database with tagged properties | Step skipped silently; all outputs still written to local filesystem (`output/`) |
| **Firecrawl** | `/enrich` (supplementary-research step), `/research` (execute-search step) | Web research via RAG: search queries per topic, extract key insights, relevance scoring, quality filtering | `/enrich`: step skipped, no web sources in study guide. `/research`: returns error message ("Firecrawl not configured") since web research is the core purpose of that workflow |

### Integration Configuration

Each integration in `tools.yaml` has:
- `enabled`: boolean toggle
- `description`: what it does
- `tools`: maps generic tool names to MCP tool names (resolved at runtime)
- `usage_notes`: setup requirements

Workflow steps check `tools.yaml` at runtime:
```
reads: [state/config/tools.yaml]
...
If ${integration}.enabled: true → execute the step
If ${integration}.enabled: false → skip silently (or return error for /research)
```

For Firecrawl, the integration is invoked as a Claude Code skill (`firecrawl:firecrawl-cli`) rather than a direct MCP tool, following its own invocation pattern.
