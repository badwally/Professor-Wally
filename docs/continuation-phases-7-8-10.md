# Professor Wally — Continuation Guide: Phases 7, 8, 10

## Context

This is a Claude Code-based AI tutoring system at `~/ai-tutor` (GitHub: https://github.com/badwally/Professor-Wally). The MVP (Phases 1–6) is complete with 56 tracked files across 5 commits. The architecture follows the [chief-of-staff](https://github.com/badwally/chief-of-staff) pattern: declarative YAML workflows, filesystem-as-database, always-on policies, and Claude Code skills.

**What's built:**
- CLAUDE.md session protocol, .gitignore, README
- 4 YAML templates (learner, course, module, tools)
- modes.yaml with operating modes and skill triggers
- 5 always-on policies (pedagogical-approach, scoring-rubric, spaced-repetition, context-discipline, anti-pattern-guards)
- 10 skill SKILL.md files (setup, ingest, enrich, study, quiz, tutor, score, review, research, switch)
- 6 workflow schemas (learner-profile, course-config, module-config, question-set, quiz-result, score-report)
- 5 prompt templates with temperature calibration (study-guide-gen, question-gen, scoring, tutoring, enrichment)
- 6 workflow definitions (setup, ingest-course, enrich-module, study-session, quiz, score-review, research-augment)
- MIT xPRO course config with 4 module configs (01-04), glossary, 10 total modules (6 unreleased)

**What's NOT built yet — this document covers these:**
- Phase 7: `/tutor` Socratic mode refinements
- Phase 8: MCP integration wiring (Notion, Gmail, Firecrawl)
- Phase 10: Documentation (architecture.md, setup-guide.md, course-authoring.md, scoring-methodology.md)

Phase 9 (test reusability with Agentic AI VP curriculum) is deferred — depends on Phases 7-8 and the VP materials at `/Users/andrewgrant/Library/Mobile Documents/com~apple~CloudDocs/Agentic AI VP Product program`.

---

## Phase 7: `/tutor` Socratic Mode Refinements

### Current State
The `/tutor` skill already exists at `skills/tutor/SKILL.md` and routes to `study-session.yaml` with a Socratic mode flag. The `study-session.yaml` workflow has a `modes` section with `standard` and `socratic` configurations, and the `section-study-loop` step has Socratic-specific instructions (elaborative interrogation, teach-back, concrete examples, interleaving).

The `prompts/tutoring.yaml` template defines the Socratic questioning framework.

### What Needs to Be Done

1. **Tutor session scoring schema** — The scoring for tutor sessions uses 5 dimensions (recall, understanding, application, integration, teaching) each scored 0-100. This is referenced in `scoring-rubric.md` and `tutoring.yaml` but needs a formal schema entry.

   **Task:** Add a `tutor-session.yaml` schema to `workflows/schemas/` with:
   ```yaml
   dimensions:
     recall: { min: 0, max: 100, description: "Can retrieve relevant facts" }
     understanding: { min: 0, max: 100, description: "Explains mechanisms, not just terms" }
     application: { min: 0, max: 100, description: "Uses concept in new context" }
     integration: { min: 0, max: 100, description: "Connects to other concepts/modules" }
     teaching: { min: 0, max: 100, description: "Explains clearly to others" }
   composite: "average of all 5 dimensions / 100"
   ```

2. **Technique rotation tracking** — The tutoring.yaml prompt says "track which techniques have been used this session — rotate, don't repeat" but there's no state mechanism for this.

   **Task:** Add a `technique_history` field to the session checkpoint in `study-session.yaml`:
   ```yaml
   checkpoint:
     # ... existing fields ...
     techniques_used: []  # e.g., ["elaborative-interrogation", "teach-back"]
   ```

3. **Cross-module interleaving questions** — The tutor mode references cross_references from module configs but doesn't have a dedicated prompt for generating interleaving questions on-the-fly.

   **Task:** Add a `prompts/interleaving.yaml` template that:
   - Takes current topic + completed module topics
   - Generates 2-3 questions that bridge concepts across modules
   - Temperature: 0.7
   - References cross_references from module configs

4. **Tutor quality feedback** — After each Socratic exchange, the tutor should provide meta-feedback on the quality of the learner's reasoning process, not just content accuracy.

   **Task:** Update `prompts/tutoring.yaml` to add a `meta_feedback` section:
   - "Your reasoning was [strong/developing] because..."
   - "Try approaching this by [technique] next time"
   - This feedback is separate from content scoring

5. **Update skill SKILL.md** — The current `skills/tutor/SKILL.md` is minimal. Flesh it out with:
   - `--topic` flag documentation
   - Socratic technique descriptions
   - Scoring dimension explanations
   - Example session flow

### Files to Create/Modify
- CREATE: `workflows/schemas/tutor-session.yaml`
- CREATE: `prompts/interleaving.yaml`
- MODIFY: `workflows/definitions/study-session.yaml` (technique tracking in checkpoint)
- MODIFY: `prompts/tutoring.yaml` (meta-feedback section)
- MODIFY: `skills/tutor/SKILL.md` (expanded documentation)

---

## Phase 8: MCP Integration Wiring

### Current State
All workflows have `optional: true` steps for MCP integrations. The `state/config/tools.template.yaml` defines three MCPs (gmail, notion, firecrawl). The `/setup` workflow asks about integrations and writes `tools.yaml`. But the actual MCP interaction logic is placeholder text — no real API calls or tool invocations.

### Available MCPs in the User's Environment
The user has these MCPs configured and available:
- **Gmail MCP** — via `mcp__ecabbfeb-21c9-43cb-8b29-dabce8f940c9__gmail_*` tools (search, read, create draft, etc.)
- **Notion MCP** — via `mcp__9e307c56-0764-4714-86e5-14547a3c3a95__notion-*` tools (search, fetch, create pages, update pages, etc.)
- **Firecrawl** — available as a skill (`firecrawl:firecrawl-cli`), handles web search, page reading, research

### What Needs to Be Done

#### 8a. Gmail/Substack Integration (for `/enrich`)

The `enrich-module.yaml` step `supplementary-research` and the `enrichment.yaml` prompt template reference Substack scanning. Wire this to actual Gmail MCP tools.

**Task:** Create `prompts/gmail-substack-scan.yaml`:
- Search query construction from `course.yaml`'s `substack_query` field and module topic keywords
- Uses `gmail_search_messages` to find newsletters
- Uses `gmail_read_message` to extract content from top 3-5 relevant results
- Extracts key insights, maps to module topics
- Output format: list of { subject, date, takeaway, relates_to_topic }

**Task:** Update `enrich-module.yaml` step `supplementary-research`:
- Check `tools.yaml` for `gmail.enabled: true`
- If enabled: construct search query from `substack_query` + module topics
- Call Gmail MCP search → read → extract
- If disabled: skip with empty output (current behavior)

**Important:** The gmail tool names in the environment are:
```
mcp__ecabbfeb-21c9-43cb-8b29-dabce8f940c9__gmail_search_messages
mcp__ecabbfeb-21c9-43cb-8b29-dabce8f940c9__gmail_read_message
```
Document these in `tools.template.yaml` so the workflow knows which tools to reference.

#### 8b. Notion Integration (for `/enrich` and `/study`)

The `enrich-module.yaml` has a `sync-to-notion` step and the `study-session.yaml` could sync session summaries. Wire to actual Notion MCP tools.

**Task:** Create `prompts/notion-sync.yaml`:
- Page creation template for study guides:
  - Title: "Module {nn}: {title} — Study Guide"
  - Tags/properties: course, module, type (study-guide | notes | session-summary)
  - Content: the markdown study guide
- Page creation template for session summaries
- Database search to avoid duplicates (check if page with same title exists)

**Task:** Update `enrich-module.yaml` step `sync-to-notion`:
- Check `tools.yaml` for `notion.enabled: true`
- If enabled and `notion_database_id` is set in course.yaml: create Notion page
- If page already exists: update instead of create
- If disabled: skip (current behavior)

**Task:** Add optional Notion sync to `study-session.yaml` after `session-summary`:
- Create a session summary page with scores, gaps, recommendations
- Link to the study guide page

**Important:** The Notion tool names in the environment are:
```
mcp__9e307c56-0764-4714-86e5-14547a3c3a95__notion-search
mcp__9e307c56-0764-4714-86e5-14547a3c3a95__notion-fetch
mcp__9e307c56-0764-4714-86e5-14547a3c3a95__notion-create-pages
mcp__9e307c56-0764-4714-86e5-14547a3c3a95__notion-update-page
```

#### 8c. Firecrawl Integration (for `/enrich` and `/research`)

The `enrich-module.yaml` and `research-augment.yaml` reference web research via Firecrawl. Wire to actual Firecrawl skill.

**Task:** Update `prompts/enrichment.yaml` to include specific Firecrawl invocation patterns:
- Use the `firecrawl:firecrawl-cli` skill for web search
- Construct search queries per topic: "{topic} explained", "{topic} applications 2025-2026", "{topic} vs {alternative}"
- Extract and summarize results (respecting copyright — short quotes only)

**Task:** Update `research-augment.yaml` step `execute-search`:
- Check `tools.yaml` for `firecrawl.enabled: true`
- If enabled: use Firecrawl skill for deep research
- Structure: search → read top results → extract insights → synthesize

**Task:** Update `tools.template.yaml` to document the Firecrawl skill name:
```yaml
firecrawl:
  enabled: false
  skill_name: "firecrawl:firecrawl-cli"
  description: "Web research — search, read pages, extract content"
```

#### 8d. Graceful Degradation Testing

**Task:** Verify each workflow handles missing MCPs correctly:
- When `tools.yaml` has `enabled: false` → step is cleanly skipped
- When MCP tool call fails at runtime → catch error, log, continue without MCP
- Workflow should never fail because an MCP is unavailable

**Task:** Add a `mcp_status` check utility in the setup workflow that tests connectivity for each enabled MCP and reports status.

### Files to Create/Modify
- CREATE: `prompts/gmail-substack-scan.yaml`
- CREATE: `prompts/notion-sync.yaml`
- MODIFY: `state/config/tools.template.yaml` (add tool names, skill references)
- MODIFY: `workflows/definitions/enrich-module.yaml` (wire MCP steps)
- MODIFY: `workflows/definitions/study-session.yaml` (add Notion sync step)
- MODIFY: `workflows/definitions/research-augment.yaml` (wire Firecrawl)
- MODIFY: `prompts/enrichment.yaml` (Firecrawl invocation patterns)

---

## Phase 10: Documentation

### Current State
The `docs/` directory has only a `.gitkeep`. The README.md covers quick start and commands but not detailed architecture or authoring guides. References to `docs/course-authoring.md` exist in the README but the file doesn't exist.

### What Needs to Be Done

#### 10a. `docs/architecture.md`

Comprehensive system architecture document covering:

1. **4-Layer Architecture** — Explain each layer with file paths:
   - L0 State: `state/` directory, YAML schemas, what's tracked vs. gitignored
   - L1 Retrieval: `indexes/` directory, concept mapping, glossary indexing
   - L2 Workflows: `workflows/definitions/`, step structure, gate patterns, input chaining
   - L3 Interface: `skills/` directory, trigger matching, modes

2. **Data Flow Diagrams** — For each major workflow:
   - `/setup` → creates learner.yaml, course.yaml, tools.yaml, initializes state
   - `/ingest` → scans materials → builds configs, glossary, indexes
   - `/enrich` → loads materials → (optional: Gmail, Firecrawl) → generates study guide + questions → seeds SRS
   - `/study` → loads state → SRS review → section study → scoring → state update
   - `/quiz` → configures assessment → administers → scores → updates progress
   - `/score` → loads all progress → computes scorecard → generates report

3. **State Management** — How state files interact:
   - progress.yaml ↔ study-session/quiz (read/write scores)
   - spaced-repetition.yaml ↔ enrich/study/review (seed/update items)
   - session-log.yaml ↔ study-session/quiz (append entries)
   - memory.yaml ↔ study-session/tutor (record insights)

4. **Scoring System** — Full explanation with formulas:
   - Composite scoring formula
   - Bloom-level breakdown methodology
   - Trend calculation (3-session window)
   - Status level thresholds

5. **Patterns from Chief-of-Staff** — Table mapping CoS patterns to tutor adaptations (this exists in the plan but should be in docs)

6. **Patterns from Open Source Research** — DeepTutor and Multi-Agent Study Assistant contributions

**Reference material:** The full architecture plan is at `/Users/andrewgrant/.claude/plans/deep-shimmying-sutherland.md` — use it as source material but restructure for documentation clarity.

#### 10b. `docs/setup-guide.md`

Step-by-step first-run walkthrough:

1. Prerequisites (Claude Code installed, MCP servers optional)
2. Clone the repo
3. Place course materials in `materials/{course-id}/`
4. Run `/setup` — what each question means, what good answers look like
5. Run `/ingest` — what it does, what to expect
6. Run `/enrich 1` — what output looks like
7. Run `/study 1` — what a session looks like
8. Common issues and troubleshooting

#### 10c. `docs/course-authoring.md`

How to onboard any new course:

1. Directory structure requirements
2. `course.yaml` schema walkthrough (reference template with annotations)
3. `module.yaml` schema walkthrough (reference template with annotations)
4. Materials organization:
   - Expected file types (.md, .pdf, .txt, .ipynb)
   - Naming conventions (handled by /ingest normalization)
   - What makes a good study guide (structure, headers, key terms)
5. Running `/ingest` and what it auto-generates
6. Manual config creation (when /ingest isn't sufficient)
7. Glossary authoring
8. Cross-references between modules
9. Example: onboarding the Agentic AI VP curriculum
   - Materials location: `/Users/andrewgrant/Library/Mobile Documents/com~apple~CloudDocs/Agentic AI VP Product program`
   - 9 modules, all markdown, existing glossary and study plans
   - How the existing structure maps to course.yaml + module configs

#### 10d. `docs/scoring-methodology.md`

Full scoring system documentation:

1. Philosophy — why these weights, why these dimensions
2. Study question scoring — binary (0/1), what counts as correct
3. Mastery question scoring — 5-dimension rubric with examples:
   - What a score of 1 vs. 3 vs. 5 looks like for each dimension
   - Sample question + sample responses at different score levels
4. Tutor session scoring — 5 dimensions (recall, understanding, application, integration, teaching)
   - What 0-25, 25-50, 50-75, 75-100 looks like for each
5. Composite formula and weights justification
6. Status levels and fluency threshold rationale
7. Bloom's taxonomy integration — how questions are distributed
8. Trend calculation methodology
9. SRS retention rate calculation

### Files to Create
- CREATE: `docs/architecture.md`
- CREATE: `docs/setup-guide.md`
- CREATE: `docs/course-authoring.md`
- CREATE: `docs/scoring-methodology.md`

---

## Build Sequence

Work these in order:

1. **Phase 7** (smallest scope) — tutor refinements, ~30 minutes
2. **Phase 8** (medium scope, most technical) — MCP wiring, ~1 hour
3. **Phase 10** (documentation) — ~45 minutes

After each phase: `git add -A && git commit` with descriptive message, then `git push`.

## Commit Convention

Follow the existing pattern:
```
Phase N: Brief summary

- Bullet points describing what changed
- Reference specific files

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

## Key Files to Read First

Before starting work, read these to understand the current system:
1. `CLAUDE.md` — session protocol and conventions
2. `state/config/modes.yaml` — skill triggers and operating modes
3. `workflows/definitions/study-session.yaml` — the main loop (most complex workflow)
4. `prompts/tutoring.yaml` — Socratic tutoring framework
5. `state/config/tools.template.yaml` — MCP integration declarations

## Remote
- GitHub: https://github.com/badwally/Professor-Wally
- Remote URL: https://github.com/badwally/Professor-Wally.git (HTTPS, not SSH)
- Branch: main
