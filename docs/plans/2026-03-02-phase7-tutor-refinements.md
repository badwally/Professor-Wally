# Phase 7: `/tutor` Socratic Mode Refinements — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Formalize tutor session scoring, add technique rotation tracking, create cross-module interleaving prompts, add persistent meta-feedback, and expand `/tutor` skill docs.

**Architecture:** All changes are declarative YAML files (schemas, prompts, workflow definitions, skill docs). No runtime code — Claude Code interprets these at session time. Follows existing patterns in `workflows/schemas/`, `prompts/`, and `skills/`.

**Tech Stack:** YAML (workflow/schema/prompt definitions), Markdown (skill docs)

---

### Task 1: Create tutor session scoring schema

**Files:**
- Create: `workflows/schemas/tutor-session.yaml`

**Step 1: Create the schema file**

```yaml
# Schema: Tutor Session Result
# Validates tutor interaction scoring from /tutor and /study (Socratic mode)
# Referenced by: scoring-rubric.md, tutoring.yaml, study-session.yaml

schema_version: "1.0"

session_fields:
  required:
    - session_id        # Unique session identifier (timestamp-based)
    - course_id
    - module_id
    - type: "tutor"
    - timestamp         # ISO 8601
    - duration_minutes
    - topics_covered: [] # Topics addressed in this session
    - mode: "socratic"

  scoring:
    dimensions:
      recall:
        min: 0
        max: 100
        description: "Can retrieve relevant facts"
        levels:
          0-25: "Cannot recall basic facts even with prompting"
          26-50: "Recalls some facts but with significant gaps or errors"
          51-75: "Recalls most facts correctly with minor gaps"
          76-100: "Recalls all relevant facts quickly and accurately"
      understanding:
        min: 0
        max: 100
        description: "Explains mechanisms, not just terms"
        levels:
          0-25: "Can only repeat definitions without explaining how or why"
          26-50: "Partial explanation; mixes up cause and effect or key mechanisms"
          51-75: "Explains core mechanisms correctly; some nuance missing"
          76-100: "Explains mechanisms fully with nuance and appropriate caveats"
      application:
        min: 0
        max: 100
        description: "Uses concept in new context"
        levels:
          0-25: "Cannot apply concept outside the exact context presented"
          26-50: "Applies concept in obvious cases but struggles with novel contexts"
          51-75: "Applies concept to new contexts with minor errors or gaps"
          76-100: "Fluently applies concept to novel and complex contexts"
      integration:
        min: 0
        max: 100
        description: "Connects to other concepts/modules"
        levels:
          0-25: "Treats concept in isolation; no connections made"
          26-50: "Makes superficial connections when prompted"
          51-75: "Identifies meaningful connections across topics or modules"
          76-100: "Spontaneously synthesizes across modules with original insight"
      teaching:
        min: 0
        max: 100
        description: "Explains clearly to others"
        levels:
          0-25: "Explanation is confusing or inaccurate"
          26-50: "Explanation covers basics but is disorganized or incomplete"
          51-75: "Clear explanation with good structure; minor gaps"
          76-100: "Excellent teach-back: clear, accurate, well-structured, uses examples"

    composite: "average of all 5 dimensions / 100"
    composite_range: { min: 0.00, max: 1.00 }

  per_interaction:
    - interaction_id    # Sequential within session
    - technique         # elaborative-interrogation | teach-back | concrete-examples | interleaving | socratic-chain
    - topic
    - dimension_scores: # Which dimensions this interaction tested
        recall: null
        understanding: null
        application: null
        integration: null
        teaching: null
    - meta_feedback:    # Reasoning quality feedback
        quality: ""     # "strong" | "developing" | "emerging"
        rationale: ""   # Why this rating
        suggestion: ""  # Technique recommendation for improvement

  meta_feedback_summary:
    reasoning_patterns: []   # Observed patterns (e.g., "skips mechanism explanations")
    effective_techniques: [] # Techniques that worked well for this learner
    growth_areas: []         # Areas where reasoning is developing

status_levels:
  below_passing: { max: 0.70, label: "Below Passing" }
  passing: { min: 0.70, max: 0.80, label: "Passing" }
  proficient: { min: 0.80, max: 0.90, label: "Proficient" }
  mastery: { min: 0.90, label: "Mastery" }
```

**Step 2: Verify schema consistency**

Check that the schema's status_levels match `quiz-result.yaml` and `score-report.yaml` (they should be identical). Check that the 5 dimensions match `scoring-rubric.md` and `tutoring.yaml`.

**Step 3: Commit**

```bash
git add workflows/schemas/tutor-session.yaml
git commit -m "Phase 7: Add tutor session scoring schema

- 5-dimension scoring model (recall, understanding, application, integration, teaching)
- Score level descriptions for each dimension (0-25, 26-50, 51-75, 76-100)
- Per-interaction records with technique tracking and meta-feedback
- Meta-feedback summary for reasoning patterns and effective techniques
- Consistent status levels with existing schemas"
```

---

### Task 2: Add technique rotation tracking to study-session.yaml

**Files:**
- Modify: `workflows/definitions/study-session.yaml` (lines 122-131, checkpoint block in section-study-loop)

**Step 1: Update the checkpoint block**

In `study-session.yaml`, find the checkpoint YAML block inside `section-study-loop` (lines 124-131) and replace it with an expanded version that includes `techniques_used`:

Find this block:
```yaml
      checkpoint:
        module_id: "${module}"
        section_index: ${current_section}
        timestamp: "${now}"
        sections_completed: ${count}
        sections_remaining: ${count}
```

Replace with:
```yaml
      checkpoint:
        module_id: "${module}"
        section_index: ${current_section}
        timestamp: "${now}"
        sections_completed: ${count}
        sections_remaining: ${count}
        techniques_used: []  # Socratic mode: tracks techniques applied this session
                             # Values: elaborative-interrogation, teach-back,
                             #         concrete-examples, interleaving, socratic-chain
```

**Step 2: Add technique rotation instruction to Socratic mode section**

In `study-session.yaml`, find the Socratic mode section (lines 115-120) and add technique rotation instructions. After line 120 (`- Rate tutor interaction on 5 dimensions...`), add:

```
      - Before selecting a technique, check checkpoint.techniques_used
      - Rotate through all 5 techniques before repeating any
      - Append each technique used to checkpoint.techniques_used
      - If resuming from checkpoint, continue rotation from where left off
```

**Step 3: Commit**

```bash
git add workflows/definitions/study-session.yaml
git commit -m "Phase 7: Add technique rotation tracking to study session

- Add techniques_used field to section-study-loop checkpoint
- Add rotation instructions to Socratic mode section
- Ensures all 5 techniques are used before repeating"
```

---

### Task 3: Create cross-module interleaving prompt template

**Files:**
- Create: `prompts/interleaving.yaml`

**Step 1: Create the prompt template**

```yaml
# Prompt Template: Cross-Module Interleaving Questions
# Used by: /tutor and /study (Socratic mode) workflows
# Temperature: 0.7

name: interleaving
description: "Generates questions that bridge concepts across modules for deeper integration"
temperature: 0.7

system_context: |
  You are an expert at designing interleaving questions — questions that force learners to
  connect concepts from different modules or topics. These questions build transfer ability
  and prevent compartmentalized understanding. Each question should require the learner to
  actively synthesize, not just recall from two separate buckets.

input_variables:
  - current_topic        # Topic currently being studied
  - current_module       # Module config for current module
  - completed_modules    # List of module configs for completed modules
  - cross_references     # From current module's cross_references field
  - learner_profile      # For adapting question style
  - progress_data        # Scores per module — target weak integration areas

prompt_template: |
  Generate 2-3 interleaving questions that bridge ${current_topic} with prior module content.

  **Current Context**
  - Module: ${current_module.id} — ${current_module.title}
  - Topic: ${current_topic}
  - Cross-references defined: ${cross_references}

  **Completed Modules Available for Interleaving**
  ${completed_modules}

  **Learner's Integration Scores**
  ${progress_data.integration_scores}

  **Question Design Rules**

  1. Each question MUST require knowledge from BOTH the current topic and a prior module.
     Bad: "What is X?" (single module recall)
     Good: "How does X from Module 2 change your understanding of Y from Module 1?"

  2. Use these connection types (vary across questions):
     - **Analogy**: "How is ${current_topic} similar to [prior concept]? Where does the analogy break down?"
     - **Contrast**: "What's the key difference between ${current_topic} and [prior concept]? Why does it matter?"
     - **Dependency**: "Why is understanding [prior concept] necessary before you can apply ${current_topic}?"
     - **Application**: "Given what you know about [prior concept], how would you use ${current_topic} to solve [scenario]?"

  3. Prioritize bridging to modules where the learner's integration score is weakest.

  4. Match question complexity to the learner's question_style preference:
     ${learner_profile.preferences.question_style}

  **Output Format**
  ```yaml
  interleaving_questions:
    - question: ""
      bridges:
        from_module: ""    # Module ID
        from_topic: ""
        to_module: ""      # Current module ID
        to_topic: ""       # Current topic
      connection_type: ""  # analogy | contrast | dependency | application
      bloom_level: ""      # analyze | evaluate | create
      expected_elements: [] # Key points a strong answer should include
  ```

output_format: yaml
```

**Step 2: Verify cross-reference format**

Read one module config (e.g., `courses/mit-xpro-ai-product/modules/module-01.yaml`) to confirm the `cross_references` field format matches what this prompt expects.

**Step 3: Commit**

```bash
git add prompts/interleaving.yaml
git commit -m "Phase 7: Add cross-module interleaving prompt template

- Generates 2-3 bridging questions connecting current topic to prior modules
- 4 connection types: analogy, contrast, dependency, application
- Prioritizes weak integration areas from progress data
- Temperature 0.7 for exploratory question generation"
```

---

### Task 4: Add meta-feedback to tutoring prompt with persistence

**Files:**
- Modify: `prompts/tutoring.yaml` (add section after scoring, update output format)
- Modify: `workflows/definitions/study-session.yaml` (update `update-state` step for meta-feedback persistence)

**Step 1: Add meta-feedback section to tutoring.yaml**

In `prompts/tutoring.yaml`, find the scoring section (lines 66-71) and add after it (before `output_format: conversational`):

```yaml

  **Meta-Feedback on Reasoning Quality**
  After each Socratic exchange, provide reasoning-quality feedback SEPARATE from content scoring:

  1. Rate reasoning quality: "strong", "developing", or "emerging"
  2. Explain WHY with specific evidence from their response:
     - Strong: "Your reasoning was strong because you identified the causal mechanism and
       anticipated the counter-argument."
     - Developing: "Your reasoning is developing — you identified the right concept but
       explained correlation rather than causation."
     - Emerging: "Your reasoning is emerging — try starting with the mechanism (why it works)
       rather than the outcome (what happens)."
  3. Suggest a technique for next time:
     - "Try approaching this by explaining the mechanism first, then the outcome."
     - "Next time, consider what would happen if the opposite were true."
     - "Try connecting this to a concept from a previous module."

  This feedback is conversational — deliver it naturally after each exchange.

  **Meta-Feedback Record** (for persistence)
  Track cumulative reasoning patterns across the session:
  ```yaml
  meta_feedback_record:
    interactions:
      - interaction_id: 1
        quality: "developing"
        rationale: "Identified concept but explained correlation not causation"
        suggestion: "Start with mechanism, then outcome"
    patterns_observed:
      - "Consistently skips mechanism explanations"
      - "Strong at generating examples"
    effective_techniques:
      - "teach-back"
    memory_candidates:  # Propose for memory.yaml
      - "Learner benefits most from teach-back technique"
      - "Tends to jump to conclusions without explaining reasoning chain"
  ```
```

Also update the output_format line:

Find:
```yaml
output_format: conversational
```

Replace with:
```yaml
output_format: conversational  # Primary output is conversational; meta_feedback_record is structured for persistence
```

**Step 2: Update study-session.yaml update-state step for meta-feedback persistence**

In `study-session.yaml`, find the `update-state` step (lines 176-200). In the prompt section, find item 3 (session-log.yaml, line 187) and item 4 (memory.yaml, lines 189-192).

Update item 3 to include meta-feedback:

Find:
```
      3. **session-log.yaml**: Append session entry with full results
```

Replace with:
```
      3. **session-log.yaml**: Append session entry with full results
         - For Socratic sessions: include meta_feedback_record from tutoring interactions
         - Record per-interaction reasoning quality ratings and technique effectiveness
```

Update item 4 to include meta-feedback memory candidates:

Find:
```
      4. **memory.yaml**: Record any learner insights discovered:
         - Patterns in what they struggle with
         - Effective teaching techniques for this learner
         - Concepts that clicked vs. didn't
```

Replace with:
```
      4. **memory.yaml**: Record any learner insights discovered:
         - Patterns in what they struggle with
         - Effective teaching techniques for this learner
         - Concepts that clicked vs. didn't
         - For Socratic sessions: persist reasoning patterns from meta_feedback_record
           - e.g., "Learner skips mechanism explanations — prompt for 'why' before 'what'"
           - e.g., "Teach-back technique is most effective for this learner"
         - Only add to memory if pattern observed across 2+ interactions (not one-offs)
```

**Step 3: Commit**

```bash
git add prompts/tutoring.yaml workflows/definitions/study-session.yaml
git commit -m "Phase 7: Add meta-feedback with persistence to tutoring

- Add reasoning-quality feedback section to tutoring.yaml (strong/developing/emerging)
- Add meta_feedback_record structure for session log persistence
- Update study-session.yaml to persist meta-feedback to session-log and memory
- Memory entries require 2+ interaction pattern confirmation"
```

---

### Task 5: Expand /tutor skill documentation

**Files:**
- Modify: `skills/tutor/SKILL.md`

**Step 1: Replace SKILL.md with expanded version**

```markdown
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
```

**Step 2: Commit**

```bash
git add skills/tutor/SKILL.md
git commit -m "Phase 7: Expand /tutor skill documentation

- Technique descriptions with examples
- Scoring dimension table and composite formula
- Meta-feedback explanation
- --topic flag documentation
- Example session flow
- Prerequisites and output details"
```

---

### Task 6: Final review and phase commit

**Step 1: Review all modified files for consistency**

Verify:
- `tutor-session.yaml` dimensions match `scoring-rubric.md` and `tutoring.yaml`
- `interleaving.yaml` input_variables match what `study-session.yaml` provides
- `tutoring.yaml` meta-feedback output matches `tutor-session.yaml` per_interaction schema
- `SKILL.md` technique names match `tutoring.yaml` technique list
- Checkpoint `techniques_used` values match technique names used everywhere else

**Step 2: Run git status and verify all changes are committed**

```bash
git status
git log --oneline -6
```

All Phase 7 changes should be across 5 commits (one per task). If any unstaged changes remain, commit them.
