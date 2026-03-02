# Phase 8 Design: MCP Integration Wiring

**Date:** 2026-03-02
**Status:** Approved

## Summary

Wire the existing optional MCP steps in workflows to concrete tool invocation patterns for Gmail, Notion, and Firecrawl. Use generic tool name placeholders — Claude matches to available MCPs at runtime. Update tools.template.yaml with tool names and skill references.

## 8a. Gmail/Substack Integration

**New file:** `prompts/gmail-substack-scan.yaml` — Prompt template (temperature 0.7) for constructing Gmail search queries from course substack_query + module topics, extracting insights from top 3-5 results, mapping to module topics. Generic tool refs: gmail_search_messages, gmail_read_message.

**Modify:** `workflows/definitions/enrich-module.yaml` — Update supplementary-research step with explicit Gmail workflow: check tools.yaml → construct query → search → read → extract.

## 8b. Notion Integration

**New file:** `prompts/notion-sync.yaml` — Two templates: study guide pages and session summary pages. Duplicate detection (search by title). Generic refs: notion_search, notion_create_pages, notion_update_page.

**Modify:** `workflows/definitions/enrich-module.yaml` — Update sync-to-notion step with structured workflow: check tools.yaml → search existing → create or update → return URL.

**Modify:** `workflows/definitions/study-session.yaml` — Add optional Notion sync step after session-summary.

## 8c. Firecrawl Integration

**Modify:** `prompts/enrichment.yaml` — Add explicit firecrawl:firecrawl-cli skill invocation pattern.

**Modify:** `workflows/definitions/research-augment.yaml` — Update execute-search step to reference firecrawl skill and check tools.yaml.

## 8d. Tools Template Update

**Modify:** `state/config/tools.template.yaml` — Add generic tool name patterns, skill references, and usage notes.

## Files

| Action | File |
|--------|------|
| CREATE | `prompts/gmail-substack-scan.yaml` |
| CREATE | `prompts/notion-sync.yaml` |
| MODIFY | `workflows/definitions/enrich-module.yaml` |
| MODIFY | `workflows/definitions/study-session.yaml` |
| MODIFY | `workflows/definitions/research-augment.yaml` |
| MODIFY | `prompts/enrichment.yaml` |
| MODIFY | `state/config/tools.template.yaml` |
