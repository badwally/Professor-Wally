# Spec Addendum: Multi-Source Academic Adapters

> Extend the research pipeline from YouTube-only to YouTube + arXiv + PubMed + Semantic Scholar.

**Date:** 2026-04-07  
**Status:** Draft  
**Depends on:** All existing stages (0–7) complete

---

## Rationale

The existing pipeline — `YouTube search → Claude semantic filter → NotebookLM corpus → Obsidian vault` — is source-agnostic from Stage 3 onward. NotebookLM ingests URLs and PDFs equally. Obsidian notes are Markdown regardless of origin. The semantic filter evaluates content quality, not content format.

Only Stages 1–2 (search + filter + staging) are YouTube-specific. Extending to academic sources means writing **source adapters** that normalize arXiv/PubMed results into the same intermediate format the filter and stager already consume. This is augmentation, not a separate tool.

---

## Architecture Change

```
CURRENT:
  src/search/youtube.py → src/filter/semantic.py → src/stage/checkpoint.py
                                                          ↓
                                               src/ingest/notebooklm.py
                                                          ↓
                                               src/output/vault.py

PROPOSED:
  src/search/youtube.py  ─→ normalize ─┐
  src/search/arxiv.py    ─→ normalize ─┤
  src/search/pubmed.py   ─→ normalize ─┼→ src/filter/semantic.py → (unchanged)
  src/search/scholar.py  ─→ normalize ─┘
```

Each adapter lives in `src/search/` alongside `youtube.py` and `queries.py`. Each implements the same interface: accept a query, return a list of normalized item dicts.

---

## 1. Common Item Schema

Replace the YouTube-only `VIDEO_REQUIRED_FIELDS` in `src/stage/schema.py` with a source-agnostic schema. The existing video fields become a subset.

### Base fields (all sources)

```python
ITEM_REQUIRED_FIELDS = {
    "item_id": str,          # e.g. "yt:dQw4w9WgXcQ", "arxiv:2401.12345", "pmid:39876543"
    "source_type": str,      # "youtube" | "arxiv" | "pubmed" | "semantic_scholar"
    "url": str,              # canonical URL
    "title": str,
    "authors": list,         # [{"name": str, "affiliation": str|None}]
    "publish_date": str,     # ISO 8601
    "description": str,      # abstract or video description (truncated to 500 chars for filter)
    "content_type": str,     # "video" | "preprint" | "journal_article" | "review" | "meta_analysis" | "conference_paper"
    "full_text_available": bool,
    "relevance_score": int,  # populated by filter
    "inclusion_rationale": str,
    "included": bool,
}
```

### Source-specific metadata

Stored in an optional `source_metadata` dict on each item. Not validated by the checkpoint schema — each adapter defines its own keys.

```python
# YouTube (existing fields, relocated)
"source_metadata": {
    "video_id": str,
    "channel_name": str,
    "channel_id": str,
    "duration": str,         # ISO 8601 duration
    "view_count": int,
    "has_captions": bool,
}

# arXiv
"source_metadata": {
    "arxiv_id": str,         # e.g. "2401.12345"
    "categories": list,      # e.g. ["cs.AI", "cs.CL"]
    "pdf_url": str,
    "primary_category": str,
    "comment": str|None,     # author comment (page count, conference acceptance)
    "journal_ref": str|None, # published version if exists
}

# PubMed
"source_metadata": {
    "pmid": str,
    "doi": str|None,
    "journal": str,
    "peer_reviewed": bool,
    "publication_type": list, # ["Journal Article", "Review", "Meta-Analysis", ...]
    "mesh_terms": list,       # MeSH controlled vocabulary
    "pmc_id": str|None,       # PubMed Central ID (full text available if present)
    "full_text_url": str|None,
}

# Semantic Scholar
"source_metadata": {
    "s2_paper_id": str,
    "doi": str|None,
    "arxiv_id": str|None,
    "venue": str,
    "citation_count": int,
    "influential_citation_count": int,
    "tldr": str|None,         # AI-generated summary
    "is_open_access": bool,
    "fields_of_study": list,
}
```

### Backward compatibility

The existing `VIDEO_REQUIRED_FIELDS` stays as-is for checkpoint validation of legacy files. New checkpoints use `ITEM_REQUIRED_FIELDS`. The `validate_checkpoint` function checks which schema to apply based on whether items have `"item_id"` (new) or `"video_id"` (legacy).

---

## 2. Source Adapters

### 2a. arXiv Adapter — `src/search/arxiv.py`

```
API:        arXiv API (REST/Atom feed, no key required)
Library:    arxiv >= 2.1.0 (pip install arxiv)
Rate limit: 1 request per 3 seconds (polite crawling)
```

**Interface:**

```python
def search_and_normalize(query: str, max_results: int = 50, **kwargs) -> list[dict]:
    """Search arXiv, return normalized items.

    kwargs:
        categories: list[str]     # e.g. ["cs.AI", "cs.CL"]
        date_from: str            # ISO date, default 2 years ago
        sort_by: str              # "relevance" | "lastUpdatedDate" | "submittedDate"
    """
```

**Query construction:**
- Claude maps research topic → arXiv category codes during domain config (stored in domain YAML)
- Queries use arXiv search syntax: `ti:` (title), `abs:` (abstract), `cat:` (category)
- Default date range: last 24 months (configurable per domain)

**Content extraction for NotebookLM (Stage 3):**
- Abstract: always available from API response
- PDF: available via `pdf_url` — NotebookLM can ingest PDF URLs directly
- No transcript extraction needed (unlike YouTube)

**Normalization:**

```python
def normalize_paper(result) -> dict:
    return {
        "item_id": f"arxiv:{result.entry_id.split('/')[-1]}",
        "source_type": "arxiv",
        "url": result.entry_id,
        "title": result.title,
        "authors": [{"name": a.name, "affiliation": None} for a in result.authors],
        "publish_date": result.published.isoformat(),
        "description": result.summary[:500],
        "content_type": "preprint",
        "full_text_available": True,  # arXiv always has PDFs
        "source_metadata": {
            "arxiv_id": result.entry_id.split("/")[-1],
            "categories": result.categories,
            "pdf_url": result.pdf_url,
            "primary_category": result.primary_category,
            "comment": result.comment,
            "journal_ref": result.journal_ref,
        },
    }
```

### 2b. PubMed Adapter — `src/search/pubmed.py`

```
API:        NCBI E-utilities (Entrez)
Library:    biopython >= 1.83 (Bio.Entrez) or direct REST
Rate limit: 3 req/sec without key, 10 req/sec with key
Auth:       NCBI API key (free, optional but recommended)
```

**Interface:**

```python
def search_and_normalize(query: str, max_results: int = 50, **kwargs) -> list[dict]:
    """Search PubMed, return normalized items.

    kwargs:
        mesh_terms: list[str]       # MeSH controlled vocabulary terms
        publication_types: list[str] # ["Review", "Meta-Analysis", "Journal Article"]
        date_from: str              # ISO date
    """
```

**Query construction:**
- Claude maps research topic → MeSH terms during domain config
- Queries use PubMed field tags: `[Title/Abstract]`, `[MeSH Terms]`, `[Author]`
- Publication type filters narrow results to substantive work

**Content extraction for NotebookLM:**
- Abstract: always available via `efetch`
- Full text: available for ~40% of articles via PubMed Central (PMC) Open Access subset
- PMC full text URL constructed from `pmc_id` when available

**API key handling:**
- Stored in `config/pubmed_api_key.txt` (gitignored, same pattern as YouTube key)
- If absent, adapter works at reduced rate (3 req/sec) — no hard failure

### 2c. Semantic Scholar Adapter — `src/search/scholar.py`

```
API:        Semantic Scholar Academic Graph API
Library:    semanticscholar >= 0.8 (pip install semanticscholar)
Rate limit: 100 req/5min without key, 1 req/sec with key (free)
Auth:       S2 API key (free, optional)
```

**Primary role:** Citation enrichment, not primary search. Used to:
1. Resolve arXiv/PubMed items to get citation counts and influential citation data
2. Find related papers via the `/recommendations` endpoint
3. Provide TLDR summaries for quick triage

**Interface:**

```python
def enrich_items(items: list[dict]) -> list[dict]:
    """Add citation data and TLDR to items that have DOI or arXiv ID."""

def find_related(item_id: str, max_results: int = 20) -> list[dict]:
    """Find related papers via S2 recommendations API."""
```

**API key handling:**
- Stored in `config/s2_api_key.txt` (gitignored)
- Fully functional without key (just slower)

---

## 3. Filter Adaptation

### `src/filter/semantic.py` changes

The `build_filter_prompt` function currently formats videos as JSON for Claude. Two changes:

**a) Source-aware prompt construction:**

The prompt template gains a source-type preamble so Claude applies the right quality signals:

```python
def build_filter_prompt(merged_policy: dict, items: list[dict]) -> str:
    # Group items by source_type
    # For each group, include source-specific evaluation criteria
    # Academic items: peer review status, citation count, methodology rigor
    # YouTube items: channel authority, production quality (existing)
```

**b) Source-specific scoring criteria in domain config:**

Extend `config/domains/*.yaml` with academic quality signals:

```yaml
quality_signals:
  # Existing YouTube signals (unchanged)
  channel_authority: { ... }
  speaker_expertise: { ... }
  content_depth: { ... }

  # New: academic paper signals
  publication_venue:
    positive_signals:
      - "Published in a peer-reviewed journal or top-tier conference"
      - "Journal with established impact in the field"
      - "Conference proceedings from CVPR, NeurIPS, ICCV, ICML, ACL, etc."
    negative_signals:
      - "Predatory journal or pay-to-publish venue"
      - "No venue information and no citation history"

  methodology_rigor:
    positive_signals:
      - "Empirical evaluation with benchmarks and baselines"
      - "Code and/or data publicly available"
      - "Clear experimental methodology and reproducibility details"
    negative_signals:
      - "Claims without supporting evidence or evaluation"
      - "No comparison to existing methods"

  citation_context:
    positive_signals:
      - "Cited by 10+ papers within 2 years of publication"
      - "Cited by influential papers in the field (via S2)"
      - "Builds on well-established prior work with clear lineage"
    negative_signals:
      - "Zero citations after 1+ year (unless very recent)"
      - "Self-citation dominated"

  cross_source:
    positive_signals:
      - "Paper explains or validates a concept covered by a YouTube source"
      - "Video provides accessible explanation of a dense paper"
      - "Source adds perspective not covered by existing corpus items"
    negative_signals:
      - "Substantially duplicates an existing corpus item from another source"
```

---

## 4. Pipeline Orchestrator Changes

### `src/pipeline.py` — extend `run_search`

```python
def run_search(
    sources: list[str],           # ["youtube", "arxiv", "pubmed"]
    domain_path: str,
    max_results_per_query: int = 50,
    # Source-specific auth
    youtube_api_key_path: str = None,
    pubmed_api_key_path: str = None,
    s2_api_key_path: str = None,
    skip_captions: bool = False,
) -> list[dict]:
    """Stage 1: Search across configured sources, return unified items."""
```

Each source adapter is called only if it appears in `sources` and has its prerequisites (API key, if required). Results are merged and deduplicated by title similarity + author overlap (academic papers may appear on both arXiv and PubMed).

### New CLI surface

Extend the existing entry points (or add a thin CLI wrapper):

```
# Existing (unchanged)
python -m src.pipeline search --domain config/domains/ai_temporal_video.yaml

# Extended
python -m src.pipeline search --domain config/domains/ai_temporal_video.yaml \
    --sources youtube,arxiv,pubmed

# Fetch specific paper by ID (bypass search, go straight to normalize + stage)
python -m src.pipeline fetch arxiv:2401.12345
python -m src.pipeline fetch doi:10.1234/example
python -m src.pipeline fetch pmid:39876543

# Citation enrichment (Semantic Scholar, post-filter)
python -m src.pipeline enrich data/staged/latest.json
```

---

## 5. Stage 3 — NotebookLM Ingestion Changes

### `src/ingest/notebooklm.py`

NotebookLM supports both URL and PDF ingestion. Changes:

- YouTube items: ingest via URL (existing behavior)
- arXiv items: ingest via `pdf_url` (NotebookLM handles PDF natively)
- PubMed items: ingest via `full_text_url` if PMC available, otherwise ingest abstract as text
- Route selection based on `source_type` field

No changes to Stages 4–5 (corpus query, deep research). NotebookLM's semantic index is source-agnostic.

---

## 6. Stage 6 — Obsidian Output Changes

### `src/output/source_notes.py`

Add a `generate_paper_note` function alongside existing `generate_source_note`:

```python
def generate_paper_note(item: dict, branch_tag: str = None, cited_in_concepts: list = None) -> str:
    """Generate Obsidian note for an academic paper."""
```

**Paper note frontmatter:**

```yaml
---
type: source-paper             # new type, distinct from "source" (video)
source_type: arxiv | pubmed
item_id: "arxiv:2401.12345"
url: "https://arxiv.org/abs/2401.12345"
title: "Paper Title"
authors: ["Author One", "Author Two"]
publish_date: "2026-03-15"
content_type: preprint | journal_article | review | meta_analysis | conference_paper
relevance_score: 4
citation_count: 42             # from S2 enrichment, null if not enriched
doi: "10.1234/example"         # null if none
tags: [topic/transformers, method/attention, field/nlp]
---
```

**Paper note body sections:**

```markdown
# {{title}}

**Authors:** {{authors}}
**Published:** {{publish_date}}
**Venue:** {{journal_ref or venue or "Preprint"}}
**DOI:** {{doi}}
**PDF:** {{pdf_url}}

## Abstract

{{description}}

## Relevance

**Score:** {{relevance_score}}/5
{{inclusion_rationale}}

## Citation Context

**Citations:** {{citation_count}} ({{influential_citation_count}} influential)
**TLDR:** {{tldr}}

## Referenced In

- [[concept-note-1]]
- [[concept-note-2]]
```

### `src/output/tags.py`

Extend the controlled vocabulary with academic-specific tag dimensions:

```python
ACADEMIC_TAG_PREFIXES = [
    "method/",        # method/transformer, method/attention, method/cnn
    "dataset/",       # dataset/imagenet, dataset/kinetics
    "venue/",         # venue/cvpr, venue/nature
    "study-type/",    # study-type/empirical, study-type/survey, study-type/theoretical
]
```

### `src/output/source_map.py`

The source map (which source contributed to which concept) needs a `source_type` field so MOC notes can distinguish and group by source type.

---

## 7. Domain Config Extension

### `config/domains/ai_temporal_video.yaml` — add academic sections

```yaml
# Existing YouTube config (unchanged)
...

# New: academic source config
academic_sources:
  arxiv:
    enabled: true
    categories: ["cs.CV", "cs.AI", "cs.LG", "cs.MM"]
    date_range_months: 24
    sort_by: relevance

  pubmed:
    enabled: false              # disabled for this domain (CV focus, not biomedical)
    mesh_terms: []
    publication_types: ["Journal Article", "Review"]
    date_range_months: 24

  semantic_scholar:
    enabled: true
    use_for: [citation_enrichment, related_papers, tldr]
    fields_of_study: ["Computer Science"]

# Search query templates (Claude populates at runtime)
query_templates:
  youtube: "{topic} {qualifier}"                        # existing
  arxiv: "ti:{topic} AND cat:{category}"               # new
  pubmed: "{topic}[Title/Abstract] AND {mesh}[MeSH]"   # new
```

---

## 8. Dependencies

Add to `requirements.txt`:

```
arxiv>=2.1.0
pymupdf>=1.24.0          # PDF text extraction (fallback if NotebookLM can't handle URL)
biopython>=1.83           # PubMed/Entrez
semanticscholar>=0.8.0    # Semantic Scholar API
```

---

## 9. New Files Summary

```
src/search/arxiv.py          # arXiv adapter
src/search/pubmed.py         # PubMed adapter
src/search/scholar.py        # Semantic Scholar adapter (enrichment)
src/search/normalize.py      # Shared normalization utilities + dedup
tests/test_arxiv.py          # arXiv adapter tests
tests/test_pubmed.py         # PubMed adapter tests
tests/test_scholar.py        # S2 adapter tests
tests/test_normalize.py      # Schema + dedup tests
config/pubmed_api_key.txt    # gitignored
config/s2_api_key.txt        # gitignored
```

**Modified files:**

```
src/pipeline.py              # Multi-source run_search
src/filter/semantic.py       # Source-aware filter prompt
src/stage/schema.py          # ITEM_REQUIRED_FIELDS + backward compat
src/ingest/notebooklm.py     # PDF URL routing
src/output/source_notes.py   # generate_paper_note
src/output/tags.py           # Academic tag prefixes
src/output/source_map.py     # source_type field
config/domains/*.yaml        # academic_sources block
requirements.txt             # new deps
.gitignore                   # pubmed_api_key.txt, s2_api_key.txt
```

---

## 10. Implementation Order

| Phase | Work | Rationale |
|-------|------|-----------|
| **A** | `ITEM_REQUIRED_FIELDS` schema + `normalize.py` shared utilities | Foundation — everything depends on the common format |
| **B** | `src/search/arxiv.py` + tests | Free API, no key, highest immediate ROI |
| **C** | Refactor `pipeline.py` for multi-source `run_search` | Wire arXiv into the existing pipeline |
| **D** | Extend `semantic.py` filter prompt for academic items | arXiv results can now be filtered |
| **E** | Extend `source_notes.py` with `generate_paper_note` | arXiv results flow to Obsidian |
| **F** | `src/search/scholar.py` (enrichment only) | Citation context makes filter + notes richer |
| **G** | `src/search/pubmed.py` + tests | Adds biomedical literature |
| **H** | Extend `notebooklm.py` for PDF URL ingestion | Academic papers enter NotebookLM corpus |
| **I** | Domain config extension + cross-source dedup | Full multi-source pipeline operational |

Phases A–E give a working arXiv→filter→Obsidian pipeline. Phases F–I layer on richness.

---

## 11. Engineering Constraints (preserved from CLAUDE.md)

- Incremental, reversible steps
- Smallest safe step
- Pragmatic over clever
- YAGNI governs scope — don't build PubMed adapter until arXiv is proven
- Surface decisions before implementing
- Domain naming, not implementation naming
- Each adapter is independently testable with mocked API responses
