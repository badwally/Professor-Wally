# Phase 8: MCP Integration Wiring — Detailed Scope

**Date:** 2026-03-02
**Status:** Implemented (8 commits, pushed to main)
**SHA range:** `03debeb..3b96ba2`

---

## What Phase 8 Does

Wires the existing placeholder `optional: true` workflow steps to concrete MCP tool invocation patterns for Gmail, Notion, and Firecrawl. Before Phase 8, every MCP step was a vague instruction like "If Notion is enabled, create a page." After Phase 8, each step has structured tool invocation sequences, prompt templates, and graceful degradation logic.

**Key design decision:** Generic tool name placeholders (e.g., `gmail_search_messages`) instead of session-specific UUID-based tool IDs (e.g., `mcp__ecabbfeb-21c9-43cb-8b29-dabce8f940c9__gmail_search_messages`). Claude matches generic names to available MCPs at runtime. This keeps the YAML portable across environments.

---

## What Was Built

### New Files (2)

#### `prompts/gmail-substack-scan.yaml` (78 lines)

Prompt template for searching Gmail Substack newsletters relevant to a module's topics.

- **Temperature:** 0.7
- **Input:** module_config, course_config, learner_profile, existing_topics
- **Search strategy:** Constructs queries from `course_config.substack_query` + module topics + learner's `substack_topics` preferences. Patterns: `from:substack ${query} ${topic}`, limited to past 6 months.
- **Tool invocation:** Two-step pattern — `gmail_search_messages` (find) → `gmail_read_message` (extract)
- **Quality filters:** Skip restated content, prefer case studies, cap at 5 articles, return empty if nothing relevant
- **Output:** `substack_scan` YAML block with `query_used`, `articles_found`, and `articles[]` each with `subject`, `date`, `author`, `takeaway`, `relates_to_topic`, `value_add`

#### `prompts/notion-sync.yaml` (98 lines)

Prompt template for syncing study materials to Notion databases.

- **Temperature:** 0.3 (deterministic for sync operations)
- **Input:** sync_type (`"study-guide"` | `"session-summary"`), course_config, module_config, content, database_id
- **Two page templates:**
  - `study_guide`: Title "Module {id}: {title} — Study Guide", properties: course/module/type/generated
  - `session_summary`: Title "Module {id}: Session Summary — {timestamp}", properties: course/module/type/composite_score/status
- **Duplicate detection:** `notion_search` by title before creating
- **Tool invocation:** Three-step — `notion_search` → `notion_update_page` (if exists) or `notion_create_pages` (if new)
- **Output:** `notion_sync` YAML with `action` (created/updated/skipped), `page_title`, `page_url`, `reason`

---

### Modified Files (5)

#### `state/config/tools.template.yaml` (19 → 39 lines)

Expanded each integration with tool name maps and usage notes:

| Integration | Additions |
|-------------|-----------|
| Gmail | `tools:` block (`search: gmail_search_messages`, `read: gmail_read_message`), `usage_notes` |
| Notion | `database_id: ""`, `tools:` block (search, create, update, fetch), `usage_notes` |
| Firecrawl | `skill_name: "firecrawl:firecrawl-cli"`, `usage_notes` |

#### `workflows/definitions/enrich-module.yaml` (~40 lines changed across 2 steps)

**Step `supplementary-research`:**
- Added `reads: [state/config/tools.yaml, prompts/enrichment.yaml, prompts/gmail-substack-scan.yaml]`
- Replaced vague prompt with 3 concrete sections:
  1. "Web Research (if firecrawl.enabled: true)" — `firecrawl:firecrawl-cli` skill invocation with query patterns per topic
  2. "Substack Scan (if gmail.enabled: true)" — `gmail_search_messages` → `gmail_read_message` workflow referencing `gmail-substack-scan.yaml`
  3. "Dynamic Topic Discovery" — flag discovered topics without adding new sections

**Step `sync-to-notion`:**
- Added `reads: [state/config/tools.yaml, prompts/notion-sync.yaml]`
- Replaced placeholder with structured Notion workflow: check tools.yaml → `notion_search` for duplicates → `notion_create_pages` or `notion_update_page` → report result
- Guards on `notion.enabled: true` AND `notion.database_id` being set

#### `workflows/definitions/study-session.yaml` (~28 lines added — new step)

**New step `sync-to-notion`** (after `session-summary`, before end of workflow):
- `optional: true`
- Creates session-summary page in Notion via `notion_create_pages`
- No duplicate check needed (each session is unique)
- Links to study guide page if one exists for the module
- Skips silently when Notion not configured (session-summary already displayed)
- Input: `${session-summary.summary}`, `${compute-scores.scores}`

#### `workflows/definitions/research-augment.yaml` (~25 lines changed across 2 steps)

**Step `execute-search`:**
- Added `reads: [state/config/tools.yaml]`
- Added comment: `# Not optional — /research requires Firecrawl; returns error message if disabled`
- If `firecrawl.enabled: true` → invoke `firecrawl:firecrawl-cli` skill
- If `firecrawl.enabled: false` → return explicit error: "Web research unavailable — Firecrawl not configured."
- **Intentionally not `optional: true`** — `/research` is meaningless without Firecrawl, so it returns a user-facing error instead of silently skipping

**Step `sync-and-report`:**
- Added `reads: [state/config/tools.yaml, prompts/notion-sync.yaml]`
- Added `database_id` guard (check `notion.enabled: true` AND `notion.database_id` is set)
- Uses `notion_create_pages` with type/topic/course properties

#### `prompts/enrichment.yaml` (4 lines added)

- **Web Search section:** Added "Invoke the firecrawl:firecrawl-cli skill" + "Check tools.yaml — only proceed if firecrawl.enabled is true"
- **Substack section:** Added "Check tools.yaml — only proceed if gmail.enabled is true" + "Use gmail_search_messages and gmail_read_message tools"

---

## Graceful Degradation Pattern

Every MCP-touching step follows a consistent pattern:

```
1. optional: true              (workflow engine won't fail if step errors)
2. reads: [tools.yaml, ...]    (load config)
3. if enabled AND configured:  (concrete tool invocation with structured instructions)
4. if disabled:                (skip with message or empty output)
```

**One exception:** `research-augment.yaml` → `execute-search` is **not optional** because `/research` is useless without Firecrawl. Returns an explicit error message instead of silently degrading.

**Guard layers:**
- `notion.enabled: true` alone is not enough — also checks `notion.database_id` is set
- `gmail.enabled: true` checked before any search queries are constructed
- `firecrawl.enabled: true` checked before skill invocation

---

## Workflow-to-MCP Integration Map

| Workflow | Step | Gmail | Notion | Firecrawl |
|----------|------|-------|--------|-----------|
| `enrich-module.yaml` | `supplementary-research` | Substack scan | — | Web research |
| `enrich-module.yaml` | `sync-to-notion` | — | Study guide sync | — |
| `study-session.yaml` | `sync-to-notion` | — | Session summary sync | — |
| `research-augment.yaml` | `execute-search` | — | — | Deep research |
| `research-augment.yaml` | `sync-and-report` | — | Research note sync | — |

**Workflows with NO MCP integration (by design):**
- `setup.yaml` — reads `tools.template.yaml` but doesn't invoke MCPs
- `ingest-course.yaml` — local file processing only
- `quiz.yaml` — assessment only, no external sync
- `score-review.yaml` — read-only progress analysis

---

## Tool Name Reference

### Gmail (2 tools)
```
gmail_search_messages    # Search emails by query string
gmail_read_message       # Read full email content by message ID
```
**Used in:** `gmail-substack-scan.yaml`, `enrichment.yaml`, `enrich-module.yaml`

### Notion (4 tools)
```
notion_search            # Search for existing pages by title/query
notion_create_pages      # Create new pages in a database
notion_update_page       # Update an existing page by ID
notion_fetch             # Fetch full page content
```
**Used in:** `notion-sync.yaml`, `enrich-module.yaml`, `study-session.yaml`, `research-augment.yaml`

### Firecrawl (skill, not tool)
```
firecrawl:firecrawl-cli  # Invoked as a Claude Code skill, not a direct MCP tool
```
**Used in:** `enrichment.yaml`, `enrich-module.yaml`, `research-augment.yaml`

---

## Commits (8)

```
03debeb Phase 8: Update tools template with tool references and usage notes
08a15d6 Phase 8: Add Gmail Substack scan prompt template
cd1b057 Phase 8: Add Notion sync prompt template
6daf190 Phase 8: Wire Gmail and Notion into enrich workflow
a2e08e2 Phase 8: Add optional Notion sync to study session
40ed58e Phase 8: Add tool invocation patterns to enrichment prompt
9308aec Phase 8: Wire Firecrawl and Notion into research workflow
3b96ba2 Phase 8: Fix research workflow consistency
```

---

## What Was NOT Built (Deferred / Out of Scope)

### From original Phase 8 spec (section 8d):

1. **`mcp_status` check utility in setup workflow** — The original spec called for a connectivity test during `/setup` that probes each enabled MCP and reports status. This was not implemented. The `setup.yaml` step `setup-tools` still only writes `enabled: true/false` to `tools.yaml` without validating that the MCP actually responds.

2. **Runtime error handling for MCP tool failures** — The current implementation checks `tools.yaml` before invoking, but does NOT have a try/catch pattern for when an MCP tool call succeeds in being dispatched but fails at runtime (e.g., Gmail API returns 401, Notion returns 404 for a bad database_id). The `optional: true` flag on the step provides some safety — the workflow won't crash — but there's no structured error logging or retry logic.

3. **Notion database_id population during `/setup`** — The `tools.template.yaml` has `database_id: ""` and `setup.yaml` asks "what database/page should notes go to?" but there's no validation or lookup flow. The learner must manually enter the correct Notion database ID. A future improvement could use `notion_search` during setup to list databases and let the learner pick one.

4. **Quiz and score-review Notion sync** — `quiz.yaml` and `score-review.yaml` have no Notion integration. Quiz results go to `output/{course-id}/quiz-results/` and score reports go to `output/{course-id}/score-report.md` but neither syncs to Notion. The original Phase 8 spec didn't call for this, but it's a natural extension.

5. **Gmail beyond Substack** — Current Gmail integration only scans `from:substack` emails. Other newsletter sources (Medium digests, LinkedIn newsletters, RSS-to-email) are not covered.

### Other gaps identified during implementation:

6. **`setup.yaml` step `setup-tools` doesn't propagate the `tools:` block structure** — The setup wizard reads `tools.template.yaml` and writes `tools.yaml`, but the prompt only says "Set `enabled: true/false` for each." It doesn't explicitly instruct writing the full `tools:` map, `database_id`, or `skill_name` fields. In practice, Claude copies the template structure, but this is implicit rather than guaranteed.

7. **No Notion sync for `/review` (inline SRS review)** — The `/review` command runs inline SRS flashcard review but has no workflow YAML — it's handled directly in the session. SRS review results update `spaced-repetition.yaml` but are not synced to Notion.

8. **Substack query field (`substack_query`) not in course template** — The `gmail-substack-scan.yaml` references `${course_config.substack_query}` but this field isn't in `courses/_template/course.template.yaml`. It comes from the learner profile's `substack_topics`. The prompt handles this gracefully by also pulling from `learner_profile.preferences.substack_topics`, but the course-level field is undocumented.

---

## How to Continue This Work

### Quick wins (under 30 min each):

**A. Add `mcp_status` probe to setup workflow**
- Add a new step `verify-tools` after `setup-tools` in `setup.yaml`
- For each enabled integration, attempt a lightweight probe:
  - Gmail: `gmail_search_messages` with a trivial query (e.g., `from:substack limit:1`)
  - Notion: `notion_search` with an empty query to verify database access
  - Firecrawl: invoke `firecrawl:firecrawl-cli` with a simple search
- Report results: "Gmail: connected / Notion: connected / Firecrawl: not responding"
- Make the step `optional: true` so setup doesn't fail if probes fail

**B. Add Notion database picker to setup**
- In `setup.yaml` step `setup-tools`, when learner says yes to Notion:
  - Use `notion_search` to list available databases
  - Present options for the learner to pick
  - Write the selected `database_id` to `tools.yaml`

**C. Add `substack_query` to course template**
- Add a `substack_query: ""` field to `courses/_template/course.template.yaml`
- Update `setup.yaml` step `setup-course-objectives` to ask about Substack search terms
- Document in `docs/course-authoring.md`

### Larger extensions:

**D. Structured MCP error handling**
- Create a `prompts/mcp-error-handling.yaml` template
- Pattern: try invocation → on failure, log to `state/progress/{course-id}/session-log.yaml` with error details → continue workflow with degraded output
- Wire into every MCP step via a `reads` reference

**E. Add Notion sync to quiz and score-review**
- `quiz.yaml`: add `sync-to-notion` step after `update-progress` — sync quiz results page
- `score-review.yaml`: add `sync-to-notion` step after `generate-report` — sync score report page
- Both follow the same pattern as `enrich-module.yaml` and `study-session.yaml`

**F. Expand Gmail beyond Substack**
- Update `gmail-substack-scan.yaml` to also search for:
  - Medium digests: `from:noreply@medium.com`
  - LinkedIn newsletters: `from:linkedin.com newsletter`
  - General keyword search without `from:substack` filter
- Rename to `gmail-newsletter-scan.yaml` or keep Substack-focused and add a separate template

---

## Key Files Quick Reference

| File | Role | Lines |
|------|------|-------|
| `state/config/tools.template.yaml` | MCP config template (copy to tools.yaml) | 39 |
| `prompts/gmail-substack-scan.yaml` | Gmail search prompt template | 78 |
| `prompts/notion-sync.yaml` | Notion sync prompt template | 98 |
| `prompts/enrichment.yaml` | RAG enrichment prompt (references Firecrawl + Gmail) | 100 |
| `workflows/definitions/enrich-module.yaml` | Enrich workflow (Gmail in supplementary-research, Notion in sync-to-notion) | 260 |
| `workflows/definitions/study-session.yaml` | Study workflow (Notion sync-to-notion step at end) | 274 |
| `workflows/definitions/research-augment.yaml` | Research workflow (Firecrawl in execute-search, Notion in sync-and-report) | 130 |
| `workflows/definitions/setup.yaml` | Setup wizard (reads tools template, writes tools.yaml) | 198 |

---

## Runtime Context

The user's environment has these MCPs available:
- **Gmail MCP:** `mcp__ecabbfeb-21c9-43cb-8b29-dabce8f940c9__gmail_*` (UUIDs are session-specific)
- **Notion MCP:** `mcp__9e307c56-0764-4714-86e5-14547a3c3a95__notion-*`
- **Firecrawl:** installed as skill `firecrawl:firecrawl-cli`

GitHub: https://github.com/badwally/Professor-Wally (HTTPS, branch: main)
