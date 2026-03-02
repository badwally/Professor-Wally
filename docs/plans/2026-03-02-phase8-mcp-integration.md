# Phase 8: MCP Integration Wiring — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Wire the existing optional MCP workflow steps to concrete tool invocation patterns for Gmail, Notion, and Firecrawl.

**Architecture:** All changes are declarative YAML (prompt templates, workflow definitions, config templates). No runtime code. Workflows already have `optional: true` on MCP steps with skip-if-disabled logic. This phase makes the "enabled" paths concrete by adding structured tool invocation instructions and prompt templates.

**Tech Stack:** YAML (workflow/prompt definitions), generic MCP tool references (Claude matches to available tools at runtime)

---

### Task 1: Update tools.template.yaml with tool references

**Files:**
- Modify: `state/config/tools.template.yaml`

**Step 1: Replace tools.template.yaml contents**

Replace the entire file with:

```yaml
# Tool Integrations — MCP Server Configuration
# Copy to tools.yaml and enable the integrations you have available
# All integrations are optional — the system degrades gracefully without them

gmail:
  enabled: false
  description: "Scan Substack newsletters for supplementary study material"
  tools:
    search: "gmail_search_messages"   # Search emails by query
    read: "gmail_read_message"        # Read full email content
  usage_notes: |
    Requires a Gmail MCP server configured in Claude Code.
    Used by /enrich to scan Substack newsletters matching course topics.
    Claude will match these generic names to the available Gmail MCP tools at runtime.

notion:
  enabled: false
  description: "Sync tagged study notes and guides to Notion databases"
  database_id: ""  # Set to your Notion database ID for study materials
  tools:
    search: "notion_search"           # Search for existing pages
    create: "notion_create_pages"     # Create new pages
    update: "notion_update_page"      # Update existing pages
    fetch: "notion_fetch"             # Fetch page content
  usage_notes: |
    Requires a Notion MCP server configured in Claude Code.
    Used by /enrich to sync study guides and by /study to sync session summaries.
    Set database_id to target a specific Notion database.
    Claude will match these generic names to the available Notion MCP tools at runtime.

firecrawl:
  enabled: false
  description: "Web research to enrich study guides with current sources"
  skill_name: "firecrawl:firecrawl-cli"
  usage_notes: |
    Requires the Firecrawl skill installed and configured.
    Used by /enrich for supplementary web research and /research for deep dives.
    Invoke via the firecrawl:firecrawl-cli skill, not as a direct MCP tool.
```

**Step 2: Commit**

```bash
git add state/config/tools.template.yaml
git commit -m "Phase 8: Update tools template with tool references and usage notes

- Add generic tool name patterns for Gmail, Notion, and Firecrawl
- Add database_id field for Notion
- Add skill_name for Firecrawl
- Add usage_notes explaining how each integration is used"
```

---

### Task 2: Create Gmail/Substack scan prompt template

**Files:**
- Create: `prompts/gmail-substack-scan.yaml`

**Step 1: Create the prompt template**

```yaml
# Prompt Template: Gmail Substack Newsletter Scan
# Used by: /enrich workflow (enrich-module.yaml, supplementary-research step)
# Temperature: 0.7

name: gmail-substack-scan
description: "Searches Gmail for Substack newsletters relevant to module topics"
temperature: 0.7

system_context: |
  You are scanning a learner's Gmail inbox for Substack newsletter content that supplements
  their course material. Extract key insights from relevant newsletters. Be selective — only
  surface articles that add genuine value beyond what the course already covers.

input_variables:
  - module_config      # Module being enriched
  - course_config      # Course config with substack_query field
  - learner_profile    # For substack_topics preferences
  - existing_topics    # Topics already in source materials

prompt_template: |
  Scan Gmail for Substack newsletters relevant to Module ${module_config.id}: ${module_config.title}.

  **Search Strategy**

  1. Construct search queries combining:
     - Course substack query: ${course_config.substack_query}
     - Module topics: ${module_config.topics}
     - Learner's Substack interests: ${learner_profile.preferences.substack_topics}

  2. Search Gmail using queries like:
     - "from:substack ${course_config.substack_query} ${topic}"
     - "from:substack ${topic} ${related_term}"
     - Limit to emails from the past 6 months for currency

  3. For each search, use the Gmail search tool to find matching emails.
     Select the top 3-5 most relevant results across all queries.

  4. For each selected email, use the Gmail read tool to extract:
     - Subject line
     - Date received
     - Author/newsletter name
     - Key takeaway (1-2 sentences summarizing the insight)
     - Which module topic it relates to
     - Why it adds value beyond the course material

  **Tool Invocation Pattern**

  Step 1 — Search:
    Tool: gmail_search_messages
    Query: constructed from above strategy
    Review results, select most relevant message IDs

  Step 2 — Read:
    Tool: gmail_read_message
    For each selected message ID from search results
    Extract content and map to module topics

  **Quality Filters**
  - Skip newsletters that merely restate course content
  - Prefer articles with practical examples, case studies, or alternative perspectives
  - Maximum 5 articles total (quality over quantity)
  - If no relevant articles found, return empty result — do not force-fit irrelevant content

  **Output Format**
  ```yaml
  substack_scan:
    query_used: ""
    articles_found: 0
    articles:
      - subject: ""
        date: ""
        author: ""
        takeaway: ""
        relates_to_topic: ""
        value_add: ""  # Why this supplements the course
  ```

output_format: yaml
```

**Step 2: Commit**

```bash
git add prompts/gmail-substack-scan.yaml
git commit -m "Phase 8: Add Gmail Substack scan prompt template

- Search query construction from course substack_query + module topics
- Generic tool references (gmail_search_messages, gmail_read_message)
- Quality filters and 5-article cap
- Structured output mapping articles to module topics"
```

---

### Task 3: Create Notion sync prompt template

**Files:**
- Create: `prompts/notion-sync.yaml`

**Step 1: Create the prompt template**

```yaml
# Prompt Template: Notion Page Sync
# Used by: /enrich workflow (sync-to-notion step) and /study workflow (optional sync)
# Temperature: 0.3 (deterministic — sync operations should be consistent)

name: notion-sync
description: "Creates or updates Notion pages for study guides and session summaries"
temperature: 0.3

system_context: |
  You are syncing study materials to Notion. Be precise with page creation and updates.
  Always check for existing pages before creating new ones to avoid duplicates.
  Structure pages cleanly with proper headings and metadata.

input_variables:
  - sync_type          # "study-guide" | "session-summary"
  - course_config      # Course config for metadata
  - module_config      # Module config for metadata
  - content            # Markdown content to sync
  - database_id        # Target Notion database ID from tools.yaml

templates:
  study_guide:
    title: "Module ${module_config.id}: ${module_config.title} — Study Guide"
    properties:
      course: "${course_config.name}"
      module: "${module_config.id}"
      type: "study-guide"
      generated: "${timestamp}"
    content: "${content}"

  session_summary:
    title: "Module ${module_config.id}: Session Summary — ${timestamp}"
    properties:
      course: "${course_config.name}"
      module: "${module_config.id}"
      type: "session-summary"
      composite_score: "${scores.composite}"
      status: "${scores.status_label}"
    content: "${content}"

prompt_template: |
  Sync content to Notion for Module ${module_config.id}: ${module_config.title}.

  **Sync Type:** ${sync_type}

  **Duplicate Detection**

  Before creating a new page:
  1. Search Notion for existing pages with matching title
     Tool: notion_search
     Query: title of the page being created
  2. If a matching page exists:
     - Update the existing page instead of creating a new one
     - Tool: notion_update_page
     - Preserve the page ID for linking
  3. If no matching page exists:
     - Create a new page
     - Tool: notion_create_pages

  **Page Creation / Update**

  For study-guide sync:
    Title: "Module ${module_config.id}: ${module_config.title} — Study Guide"
    Properties: course, module, type (study-guide), generated date
    Body: The full study guide markdown content

  For session-summary sync:
    Title: "Module ${module_config.id}: Session Summary — ${timestamp}"
    Properties: course, module, type (session-summary), composite score, status
    Body: Session results, scores, gaps, recommendations

  **Tool Invocation Pattern**

  Step 1 — Check for duplicates:
    Tool: notion_search
    Query: page title

  Step 2a — Update existing page (if found):
    Tool: notion_update_page
    Page ID: from search results
    Content: updated markdown

  Step 2b — Create new page (if not found):
    Tool: notion_create_pages
    Database ID: ${database_id}
    Title and properties from template above
    Content: markdown body

  **Output Format**
  ```yaml
  notion_sync:
    action: "created" | "updated" | "skipped"
    page_title: ""
    page_url: ""  # Notion page URL if available
    reason: ""    # If skipped, why
  ```

output_format: yaml
```

**Step 2: Commit**

```bash
git add prompts/notion-sync.yaml
git commit -m "Phase 8: Add Notion sync prompt template

- Two templates: study-guide and session-summary pages
- Duplicate detection via notion_search before creating
- Generic tool references (notion_search, notion_create_pages, notion_update_page)
- Structured output with action, page title, and URL"
```

---

### Task 4: Wire Gmail and Notion into enrich-module.yaml

**Files:**
- Modify: `workflows/definitions/enrich-module.yaml`

**Step 1: Update the supplementary-research step**

In `enrich-module.yaml`, find the `supplementary-research` step (lines 64-85). Replace the entire prompt block with a version that has concrete Gmail and Firecrawl invocation:

Find:
```yaml
    prompt: |
      If Firecrawl is enabled: Search for 3-5 supplementary sources per major topic.
      If Gmail is enabled: Scan for relevant Substack articles.

      Follow the enrichment.yaml prompt template for search strategy.

      Quality filters:
      - Recent (within 2 years unless foundational)
      - Authoritative (.edu, major publications, recognized experts)
      - Adds perspective not already in course materials
      - Maximum 5 sources per topic

      If neither tool is enabled, skip this step with empty output.
```

Replace with:
```yaml
    reads: [state/config/tools.yaml, prompts/enrichment.yaml, prompts/gmail-substack-scan.yaml]
    prompt: |
      Gather supplementary material from available external sources.
      Load tools.yaml to check which integrations are enabled.

      **1. Web Research (if firecrawl.enabled: true)**
      Follow the enrichment.yaml prompt template's "Web Search (Firecrawl)" section.
      Invoke the firecrawl:firecrawl-cli skill for each search query.
      Search queries per topic:
      - "${topic} explained" for foundational understanding
      - "${topic} applications ${current_year}" for currency
      - "${topic} vs ${alternative}" for comparative understanding
      Select 3-5 best sources per topic with relevance scoring.

      **2. Substack Scan (if gmail.enabled: true)**
      Follow the gmail-substack-scan.yaml prompt template.
      Use gmail_search_messages to find newsletters matching:
      - Course substack_query + module topic keywords
      - Learner's substack_topics preferences
      Use gmail_read_message to extract content from top 3-5 results.
      Map each article to module topics.

      **3. Dynamic Topic Discovery**
      If research surfaces important related topics NOT in the module config:
      - Flag as "Discovered Topic" with relevance and suggested placement
      - Do NOT add entire new sections

      **If neither tool is enabled**, skip with empty output — this is normal.

      Quality filters:
      - Recent (within 2 years unless foundational)
      - Authoritative (.edu, major publications, recognized experts)
      - Adds perspective not already in course materials
      - Maximum 5 sources per topic
```

**Step 2: Update the sync-to-notion step**

Find the `sync-to-notion` step (lines 177-194). Replace the prompt block:

Find:
```yaml
    prompt: |
      If Notion is enabled in tools.yaml:
      1. Create a Notion page for the study guide with tags:
         - Course: ${course-id}
         - Module: ${module-id}
         - Type: study-guide
      2. Include the study guide content in markdown format
      3. Add a link back to the local file path

      If Notion is not enabled, skip with message "Notion sync skipped — not configured."
```

Replace with:
```yaml
    reads: [state/config/tools.yaml, prompts/notion-sync.yaml]
    prompt: |
      Sync the generated study guide to Notion.
      Load tools.yaml to check if notion.enabled is true.

      **If notion.enabled: true AND notion.database_id is set:**
      Follow the notion-sync.yaml prompt template with sync_type: "study-guide".

      1. Search for existing page:
         Tool: notion_search
         Query: "Module ${module-id}: ${module_title} — Study Guide"

      2. If page exists → update it:
         Tool: notion_update_page
         Update content with latest study guide markdown

      3. If page does not exist → create it:
         Tool: notion_create_pages
         Database: ${notion.database_id}
         Title: "Module ${module-id}: ${module_title} — Study Guide"
         Properties: course, module, type (study-guide), generated date
         Body: study guide markdown content

      4. Report sync result (created/updated/skipped + page URL)

      **If notion.enabled: false or database_id not set:**
      Skip with message "Notion sync skipped — not configured."
```

**Step 3: Commit**

```bash
git add workflows/definitions/enrich-module.yaml
git commit -m "Phase 8: Wire Gmail and Notion into enrich workflow

- Update supplementary-research with concrete Firecrawl and Gmail invocations
- Update sync-to-notion with structured Notion workflow (search → create/update)
- Add reads references for tools.yaml and new prompt templates
- Graceful skip when tools not enabled"
```

---

### Task 5: Add Notion sync step to study-session.yaml

**Files:**
- Modify: `workflows/definitions/study-session.yaml`

**Step 1: Add sync-to-notion step after session-summary**

In `study-session.yaml`, after the `session-summary` step (which ends at line 244 with `gate: display`), add a new step:

```yaml

  - name: sync-to-notion
    executor: prompt
    optional: true  # Skipped if Notion not enabled
    reads: [state/config/tools.yaml, prompts/notion-sync.yaml]
    prompt: |
      Sync session summary to Notion.
      Load tools.yaml to check if notion.enabled is true.

      **If notion.enabled: true AND notion.database_id is set:**
      Follow the notion-sync.yaml prompt template with sync_type: "session-summary".

      1. Create a session summary page (no duplicate check needed — each session is unique):
         Tool: notion_create_pages
         Database: ${notion.database_id}
         Title: "Module ${module-id}: Session Summary — ${timestamp}"
         Properties: course, module, type (session-summary), composite score, status
         Body: session results including scores, gaps, and recommendations

      2. If a study guide page exists for this module, add a link to it.

      **If notion.enabled: false or database_id not set:**
      Skip silently — no message needed since session-summary already displayed.
    input:
      summary: ${session-summary.summary}
      scores: ${compute-scores.scores}
    output:
      type: text
      key: notion_sync
```

**Step 2: Commit**

```bash
git add workflows/definitions/study-session.yaml
git commit -m "Phase 8: Add optional Notion sync to study session

- New sync-to-notion step after session-summary
- Creates session summary page with scores and recommendations
- Links to study guide page if it exists
- Skips silently when Notion not configured"
```

---

### Task 6: Update enrichment.yaml with Firecrawl invocation pattern

**Files:**
- Modify: `prompts/enrichment.yaml`

**Step 1: Add Firecrawl skill reference to the Web Search section**

In `enrichment.yaml`, find the "Web Search (Firecrawl)" section (line 34). Add a tool invocation note after "For each major topic, search for:" (line 35).

Find:
```yaml
  1. **Web Search (Firecrawl)**
     For each major topic, search for:
```

Replace with:
```yaml
  1. **Web Search (Firecrawl)**
     Invoke the firecrawl:firecrawl-cli skill for web research.
     Check tools.yaml — only proceed if firecrawl.enabled is true.
     For each major topic, search for:
```

**Step 2: Add tool invocation note to the Substack section**

Find:
```yaml
  2. **Substack Scanning (Gmail MCP)**
     If enabled, search for newsletters matching:
```

Replace with:
```yaml
  2. **Substack Scanning (Gmail MCP)**
     Check tools.yaml — only proceed if gmail.enabled is true.
     Use gmail_search_messages and gmail_read_message tools.
     Search for newsletters matching:
```

**Step 3: Commit**

```bash
git add prompts/enrichment.yaml
git commit -m "Phase 8: Add tool invocation patterns to enrichment prompt

- Reference firecrawl:firecrawl-cli skill in web search section
- Reference Gmail MCP tools in Substack scanning section
- Add tools.yaml check instructions for both"
```

---

### Task 7: Wire Firecrawl into research-augment.yaml

**Files:**
- Modify: `workflows/definitions/research-augment.yaml`

**Step 1: Update the execute-search step**

In `research-augment.yaml`, find the `execute-search` step (lines 34-57). Update the prompt:

Find:
```yaml
    prompt: |
      Execute web research using Firecrawl:
```

Replace with:
```yaml
    reads: [state/config/tools.yaml]
    prompt: |
      Execute web research.
      Load tools.yaml to check if firecrawl.enabled is true.

      **If firecrawl.enabled: true:**
      Invoke the firecrawl:firecrawl-cli skill for each search query.

      **If firecrawl.enabled: false:**
      Report: "Web research unavailable — Firecrawl not configured.
      Enable Firecrawl in tools.yaml to use /research."
      Return empty results.

      **Research execution (when enabled):**
```

**Step 2: Update the sync-and-report step for Notion**

Find the `sync-and-report` step (lines 91-110). Update the prompt:

Find:
```yaml
    prompt: |
      If Notion is enabled:
      - Create Notion page with tags: research, ${topic}, ${course-id}
```

Replace with:
```yaml
    reads: [state/config/tools.yaml, prompts/notion-sync.yaml]
    prompt: |
      If Notion is enabled (notion.enabled: true in tools.yaml):
      - Use notion_create_pages to create a page
      - Database: ${notion.database_id}
      - Tags/properties: type (research), topic (${topic}), course (${course-id})
      - Body: the research note markdown content
```

**Step 3: Commit**

```bash
git add workflows/definitions/research-augment.yaml
git commit -m "Phase 8: Wire Firecrawl and Notion into research workflow

- Reference firecrawl:firecrawl-cli skill in execute-search step
- Add tools.yaml check with graceful error when Firecrawl not configured
- Wire Notion sync with concrete tool references in sync-and-report step"
```

---

### Task 8: Final review and consistency check

**Step 1: Verify tool name consistency**

Check that these generic tool names are used consistently across all files:
- `gmail_search_messages` and `gmail_read_message` — in tools.template.yaml, gmail-substack-scan.yaml, enrichment.yaml, enrich-module.yaml
- `notion_search`, `notion_create_pages`, `notion_update_page`, `notion_fetch` — in tools.template.yaml, notion-sync.yaml, enrich-module.yaml, study-session.yaml, research-augment.yaml
- `firecrawl:firecrawl-cli` — in tools.template.yaml, enrichment.yaml, enrich-module.yaml, research-augment.yaml

**Step 2: Verify graceful degradation pattern**

Each workflow step that uses an MCP should follow this pattern:
1. Has `optional: true` on the step
2. Checks `tools.yaml` for `enabled: true`
3. If disabled: skips cleanly with appropriate message or empty output
4. If enabled: invokes tools with structured instructions

**Step 3: Run git status and verify all changes committed**

```bash
git status
git log --oneline -9
```
