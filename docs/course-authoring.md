# Course Authoring Guide

This system is course-agnostic. Any subject -- from machine learning to music theory to
foreign policy -- can be onboarded by providing source materials and a small amount of
YAML configuration. This guide walks through every step of that process.

---

## 1. Directory Structure

A course requires two directories: a **configuration directory** under `courses/` and a
**materials directory** under `materials/`.

```
courses/{course-id}/
  course.yaml              # Course-level config
  glossary.yaml            # Term definitions
  modules/
    01-module-name.yaml    # Per-module config
    02-module-name.yaml
    ...

materials/{course-id}/
  mod-01/
    study-guide.md         # Primary content (markdown preferred)
    transcripts/           # Optional video/lecture transcripts
    quick-ref.pdf          # Optional reference sheets
    workbook.txt           # Optional exercises
  mod-02/
    ...
```

The `{course-id}` must be a URL-safe slug (lowercase, hyphens, no spaces).
Examples: `mit-ai-xpro`, `intro-to-statistics`, `product-management-101`.

---

## 2. `course.yaml` Schema

Copy the template from `courses/_template/course.template.yaml` into your course
directory and fill in each field.

```yaml
id: "intro-to-statistics"              # Unique course slug
name: "Introduction to Statistics"     # Human-readable display name
provider: "Khan Academy"               # Course provider or author
version: "2026-Q1"                     # Version tag for tracking updates

objectives:
  - text: "Apply hypothesis testing to real-world datasets"
    measurable: "Score >= 80% on hypothesis testing module"
    bloom_level: apply     # remember | understand | apply | analyze | evaluate | create

success_criteria:
  - criterion: "Score >= 80% composite on all modules"
    threshold: 0.80
    module_scope: "all"    # "all" or a specific module id

materials_path: "materials/intro-to-statistics"   # Relative from repo root

modules:
  - id: "01"
    config: "modules/01-descriptive-stats.yaml"
  - id: "02"
    config: "modules/02-probability.yaml"
  - id: "03"
    config: "modules/03-hypothesis-testing.yaml"
    status: unreleased     # Omit or set to "unreleased" for future modules

scoring:
  passing_threshold: 0.70    # 70% minimum to pass
  mastery_threshold: 0.90    # 90% for mastery designation
  fluency_threshold: 0.80    # 80% required to advance to next module
  weights:
    study_questions: 0.20    # 20% from study session questions
    mastery_questions: 0.50  # 50% from quiz/mastery assessments
    tutor_sessions: 0.30     # 30% from Socratic tutoring quality

glossary: "glossary.yaml"   # Path relative to course directory

# Optional MCP integrations (leave empty if not used)
substack_query: ""           # Gmail search query for related newsletters
notion_database_id: ""       # Notion database ID for notes sync
```

### Field details

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique slug used in file paths and progress tracking. |
| `name` | Yes | Display name shown in session summaries. |
| `provider` | Yes | Source institution, author, or "Self-authored". |
| `version` | Yes | Version tag for tracking curriculum updates. |
| `objectives` | Yes | Course-level learning objectives with Bloom taxonomy levels. |
| `success_criteria` | Yes | Concrete criteria defining what "done" means. |
| `materials_path` | Yes | Relative path to the materials directory. |
| `modules` | Yes | Ordered list of module IDs and config paths. |
| `scoring` | Yes | Thresholds and composite score weights. |
| `glossary` | Yes | Path to the course glossary file. |
| `substack_query` | No | Gmail search for Substack newsletters (requires Gmail MCP). |
| `notion_database_id` | No | Notion database for notes sync (requires Notion MCP). |

---

## 3. `module.yaml` Schema

Each module gets its own config file. Copy from `courses/_template/module.template.yaml`
or let `/ingest` generate them automatically.

```yaml
id: "01"
title: "Descriptive Statistics"
description: "Measures of central tendency, spread, and distribution shape"
sequence: 1                              # 1-based order in the course

learning_outcomes:
  - outcome: "Calculate and interpret mean, median, and mode"
    bloom_level: apply
    key_concepts: [mean, median, mode, central-tendency]
  - outcome: "Evaluate which measure of center is appropriate for skewed data"
    bloom_level: evaluate
    key_concepts: [skewness, outliers, robustness]

topics:
  - name: "Measures of Central Tendency"
    subtopics:
      - "Arithmetic mean and weighted mean"
      - "Median for skewed distributions"
      - "Mode for categorical data"
    key_terms: [mean, median, mode, weighted-mean]
    source_files: ["study-guide.md"]     # Relative to materials/{course-id}/mod-01/

prerequisites: []                         # Module IDs that should be completed first

estimated_hours: 4                        # Rough estimate of study time

materials:
  study_guide: "mod-01/study-guide.md"   # Relative to materials/{course-id}/
  transcripts:
    - "mod-01/lecture-01.pdf"
  quick_reference: "mod-01/quick-ref.pdf"
  workbook: "mod-01/workbook.txt"
  supplementary: []

mastery_indicators:
  - indicator: "Can explain when median is preferred over mean"
    evidence: "Gives correct reasoning about outlier sensitivity with examples"
  - indicator: "Can calculate standard deviation by hand for small datasets"
    evidence: "Arrives at correct answer and explains each step"

cross_references:
  - module_id: "02"
    relationship: "builds-on"            # builds-on | contrasts-with | applies-to
    specific_topics: ["Probability distributions extend descriptive statistics"]
```

### Field details

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Two-digit module identifier (e.g. `"01"`). |
| `title` | Yes | Module display name. |
| `description` | Yes | One-sentence summary of the module scope. |
| `sequence` | Yes | Numeric position in the course (1-based). |
| `learning_outcomes` | Yes | What the learner will demonstrate, with Bloom levels. |
| `topics` | Yes | Hierarchical topic list with subtopics, key terms, and source files. |
| `prerequisites` | No | List of module IDs that should be completed first. |
| `estimated_hours` | No | Expected study time. |
| `materials` | Yes | Paths to study guide, transcripts, reference sheets, workbooks. |
| `mastery_indicators` | Yes | Observable behaviors that demonstrate mastery. |
| `cross_references` | No | Links to related modules for interleaving questions. |

### Bloom levels

Assign a Bloom level to each learning outcome and glossary term. The system uses
these to calibrate question difficulty and track progression.

| Level | Verb examples | Question style |
|-------|--------------|----------------|
| `remember` | define, list, recall | "What are the four stages of..." |
| `understand` | explain, describe, summarize | "In your own words, explain why..." |
| `apply` | calculate, implement, use | "Given this dataset, compute..." |
| `analyze` | compare, differentiate, examine | "What are the trade-offs between..." |
| `evaluate` | justify, critique, assess | "Which approach is best for this scenario and why?" |
| `create` | design, propose, construct | "Design a system that..." |

---

## 4. Materials Organization

### Supported file types

| Type | Role | Notes |
|------|------|-------|
| `.md` | Study guides (primary) | Preferred format. Loaded directly into context. |
| `.pdf` | Transcripts, reference guides | Referenced but not loaded into context. |
| `.txt` | Workbooks, exercises | Loaded when needed for specific activities. |
| `.ipynb` | Jupyter notebooks | Used for coding assignments. |

Markdown is the preferred format for primary study content because it can be read
directly into the LLM context window. PDFs are catalogued and referenced but their
content must be summarized in markdown for effective tutoring.

### Naming conventions

The `/ingest` workflow handles common naming inconsistencies automatically:

- `Mod 1`, `Module 1`, `module 1`, `mod-01` all normalize to module `01`
- Folder names are preserved in `materials/` but mapped to standardized module IDs
- File names within folders are referenced as-is in module configs

### What makes a good study guide

- Clear hierarchical headers (`##` for topics, `###` for subtopics)
- Key terms in **bold** at first occurrence
- Structured sections: overview, concepts, examples, summary
- Definitions close to where terms are introduced
- Case studies and practical examples clearly labeled

---

## 5. Running `/ingest`

The `/ingest` workflow automates most of the configuration work. Run it after placing
materials in the `materials/{course-id}/` directory.

```
/ingest {course-id}
```

### What `/ingest` does

1. **Validates materials**: Reads `course.yaml` for `materials_path`, verifies the
   directory exists, lists all subdirectories.
2. **Normalizes structure**: Maps folder names (e.g. "Mod 1", "Module 3") to
   standardized module IDs.
3. **Extracts module structure**: Identifies file types in each module directory.
   The largest `.md` file becomes the primary study guide. Builds a materials
   manifest mapping every file to its role.
4. **Extracts topics and terms**: Reads each study guide to extract main topics
   (from `##` headers), subtopics (from `###` headers), key terms (bolded words
   and definitions), learning outcomes, and cross-references.
5. **Generates module configs**: Creates a `module.yaml` for each module following
   the template schema, with Bloom levels assigned to learning outcomes.
6. **Generates glossary**: Builds `glossary.yaml` with all discovered terms,
   organized alphabetically, with module sources and related terms.
7. **Builds retrieval indexes**: Creates `concepts.yaml` (concept-to-module mapping),
   `glossary-index.yaml` (unified term lookup), and `recent.yaml` (ingest metadata)
   in the `indexes/` directory.
8. **Updates course config**: Adds discovered modules to `course.yaml` with correct
   IDs and config paths.

### Incremental ingest

By default, `/ingest` skips modules that have already been indexed. Use `--force` to
re-index everything. This is useful when materials have been updated.

### What to verify after ingest

- Module configs exist in `courses/{course-id}/modules/` for each module
- `glossary.yaml` is populated with discovered terms
- Topics in each module config reflect the actual study guide content
- Modules flagged as "needs enrichment" have no study guide and need manual preparation
- `materials` paths in module configs point to the correct files

---

## 6. Manual Config Creation

Sometimes `/ingest` is not the right starting point. Create configs manually when:

- **Materials are unstructured**: Content lacks clear headers or consistent formatting.
  Write the topic hierarchy yourself to impose structure.
- **You want specific topic groupings**: The auto-extracted topics do not match how
  you want to organize the curriculum.
- **You need custom cross-references**: You know specific relationships between modules
  that may not be obvious from content alone.
- **You want to control Bloom level distribution**: You want to ensure a specific
  progression from `remember` through `create` across the course.
- **Materials are not yet available**: You want to define the course structure first
  and add materials later.

To create configs manually:

1. Copy `courses/_template/course.template.yaml` to `courses/{course-id}/course.yaml`
2. Copy `courses/_template/module.template.yaml` for each module
3. Fill in all required fields following the schemas above
4. Create an empty `glossary.yaml` with just the `course_id` field
5. Run `/ingest` later to fill in anything you left incomplete

---

## 7. Glossary Authoring

The glossary is a YAML file mapping terms to definitions, organized alphabetically.

```yaml
course_id: "intro-to-statistics"

terms:
  - term: "Confidence Interval"
    definition: "A range of values that is likely to contain the true population parameter with a specified level of confidence."
    module_sources: ["03", "04"]
    related_terms: ["margin-of-error", "significance-level"]
    bloom_level: understand

  - term: "Standard Deviation"
    definition: "A measure of the amount of variation in a dataset, calculated as the square root of the variance."
    module_sources: ["01"]
    related_terms: ["variance", "spread"]
    bloom_level: apply
```

### Glossary conventions

- **Auto-generated** during `/ingest` from bolded terms and definitions in study guides.
- **Manually editable** after generation -- add, remove, or refine entries as needed.
- **Referenced during enrichment**: The `/enrich` workflow uses the glossary to ensure
  consistent terminology in generated study materials and questions.
- **Cross-module tracking**: The `module_sources` field tracks every module where a term
  appears, enabling interleaving during review.

---

## 8. Cross-References Between Modules

Cross-references connect related content across modules. They are defined in each
module's `cross_references` field.

```yaml
cross_references:
  - module_id: "02"
    relationship: "builds-on"
    specific_topics: ["Probability extends descriptive statistics concepts"]
  - module_id: "05"
    relationship: "applies-to"
    specific_topics: ["Hypothesis testing applies inferential methods to experiments"]
```

### Relationship types

| Type | Meaning | Used for |
|------|---------|----------|
| `builds-on` | This module extends concepts from the referenced module. | Prerequisites, scaffolding. |
| `contrasts-with` | This module covers an alternative approach to the same problem. | Compare-and-contrast questions. |
| `applies-to` | Concepts from this module are applied in the referenced module. | Transfer and synthesis questions. |

Cross-references drive interleaving questions during Socratic tutoring. When studying
Module 3, the tutor may ask a question that connects back to Module 1 concepts if a
cross-reference exists between them.

---

## 9. Example: Onboarding a New Curriculum

Suppose you have a 9-module data science curriculum with markdown study guides, a
glossary, and study plans. Here is how to onboard it.

### Step 1: Create the course directory

```bash
mkdir -p courses/data-science-fundamentals/modules
```

### Step 2: Place materials

Copy or symlink your materials into the expected location:

```bash
mkdir -p materials/data-science-fundamentals
cp -r /path/to/your/materials/* materials/data-science-fundamentals/
```

After copying, the materials directory might look like:

```
materials/data-science-fundamentals/
  Module 1 - Intro to Data Science/
    study-guide.md
    transcript-01.pdf
  Module 2 - Data Wrangling/
    study-guide.md
    exercises.txt
  Module 3 - Exploratory Analysis/
    study-guide.md
  ...
```

The inconsistent folder names ("Module 1", "Module 2") are fine -- `/ingest` will
normalize them.

### Step 3: Create a minimal `course.yaml`

```yaml
id: "data-science-fundamentals"
name: "Data Science Fundamentals"
provider: "Self-authored"
version: "2026-Q1"

objectives:
  - text: "Apply the full data science workflow from data collection to presentation"
    measurable: "Complete end-to-end analysis project"
    bloom_level: apply

success_criteria:
  - criterion: "Score >= 80% composite on all modules"
    threshold: 0.80
    module_scope: "all"

materials_path: "materials/data-science-fundamentals"

modules: []     # Will be populated by /ingest

scoring:
  passing_threshold: 0.70
  mastery_threshold: 0.90
  fluency_threshold: 0.80
  weights:
    study_questions: 0.20
    mastery_questions: 0.50
    tutor_sessions: 0.30

glossary: "glossary.yaml"
```

### Step 4: Run `/ingest`

```
/ingest data-science-fundamentals
```

This will scan all 9 module directories, generate module configs, build the glossary,
and update `course.yaml` with the discovered modules.

### Step 5: Verify the output

Check that:
- 9 module YAML files exist in `courses/data-science-fundamentals/modules/`
- `glossary.yaml` contains terms extracted from your study guides
- Each module config has topics, learning outcomes, and correct materials paths
- The `course.yaml` modules list now has 9 entries

### Step 6: Enrich the first module

```
/enrich 1
```

This generates an enriched study guide with active recall questions, mastery questions,
and Socratic prompts for Module 1. Repeat for each module as you are ready to study it.

---

## 10. Tips and Troubleshooting

- **Start with `/ingest` even if materials are imperfect.** You can always edit the
  generated configs afterward. It is faster to correct auto-generated YAML than to
  write it from scratch.
- **One study guide per module.** If a module has multiple markdown files, consolidate
  them or designate one as primary. The largest `.md` file is chosen automatically.
- **Glossary terms propagate.** Terms added to the glossary appear in generated
  questions and study guides. Keep definitions concise and precise.
- **Bloom levels matter.** The system generates questions at the Bloom level specified
  for each learning outcome. If all outcomes are set to `remember`, you will only get
  recall questions. Distribute levels across the taxonomy for a balanced assessment.
- **Cross-references are optional but valuable.** Even two or three cross-references
  per module significantly improve the quality of interleaving questions during tutoring.
