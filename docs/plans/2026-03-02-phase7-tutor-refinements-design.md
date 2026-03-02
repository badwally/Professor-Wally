# Phase 7 Design: `/tutor` Socratic Mode Refinements

**Date:** 2026-03-02
**Status:** Approved

## Summary

Five refinements to the Socratic tutoring mode: formalize scoring schema, add technique rotation tracking, create cross-module interleaving prompt, add meta-feedback with persistence, and expand skill documentation.

## 1. Tutor Session Scoring Schema

**New file:** `workflows/schemas/tutor-session.yaml`

Formalizes the 5-dimension scoring model (recall, understanding, application, integration, teaching) currently referenced informally in `scoring-rubric.md` and `tutoring.yaml`. Follows existing schema patterns (like `quiz-result.yaml`). Includes per-interaction records and composite calculation formula.

## 2. Technique Rotation Tracking

**Modify:** `workflows/definitions/study-session.yaml`

Add `techniques_used: []` to the checkpoint block in `section-study-loop`. Add instruction in Socratic mode section to append each technique as used and consult the list before selecting the next.

## 3. Cross-Module Interleaving Prompt

**New file:** `prompts/interleaving.yaml`

Prompt template (temperature 0.7) taking current topic + completed module topics + cross_references from module configs. Generates 2-3 bridging questions with connection types (analogy, contrast, dependency, application).

## 4. Meta-Feedback with Persistence

**Modify:** `prompts/tutoring.yaml`

Add `meta_feedback` section generating reasoning-quality feedback ("Your reasoning was strong/developing because...") and technique suggestions. Output includes `meta_feedback_record` structure for:
- Appending to session-log.yaml under `meta_feedback` field
- Surfacing patterns in memory.yaml (e.g., "learner consistently skips mechanism explanations")

**Modify:** `workflows/definitions/study-session.yaml`

Update `update-state` step to persist meta-feedback to session log and write reasoning patterns to memory.

## 5. Expanded Skill Documentation

**Modify:** `skills/tutor/SKILL.md`

Add `--topic` flag docs, Socratic technique descriptions, scoring dimension explanations, example session flow.

## Files

| Action | File |
|--------|------|
| CREATE | `workflows/schemas/tutor-session.yaml` |
| CREATE | `prompts/interleaving.yaml` |
| MODIFY | `workflows/definitions/study-session.yaml` |
| MODIFY | `prompts/tutoring.yaml` |
| MODIFY | `skills/tutor/SKILL.md` |
