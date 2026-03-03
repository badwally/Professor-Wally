# Scoring Methodology

This document defines the complete scoring system used by the AI tutor. Every assessment, whether a quick study question or a full tutoring session, feeds into a unified composite score that tracks learner progress toward mastery.

---

## 1. Philosophy

The scoring system is built around a core insight: **knowing facts is necessary but insufficient for mastery**. True understanding requires the ability to reason about concepts, apply them in novel contexts, and explain them to others.

The system measures three distinct dimensions of learning:

| Dimension | Weight | What It Captures |
|-----------|--------|-----------------|
| Study questions | 20% | Factual recall and basic comprehension |
| Mastery questions | 50% | Deep understanding, analysis, and synthesis |
| Tutor sessions | 30% | Reasoning ability and transferable skill |

**Why these weights?** Mastery-level understanding receives the largest weight (50%) because the ability to analyze, evaluate, and create demonstrates genuine learning that transfers beyond the course. Study questions carry only 20% because factual recall, while foundational, can mask shallow understanding. Tutor sessions at 30% capture something neither question type can: the ability to reason through problems in real time, connect ideas across modules, and teach concepts to others, which is the strongest evidence of deep learning.

---

## 2. Study Question Scoring

Study questions target factual recall and basic comprehension. They are scored **binary**: correct (1.0) or incorrect (0.0). Partial credit is not awarded.

### Bloom's Level Distribution

| Bloom Level | Proportion | Example |
|-------------|-----------|---------|
| Remember | 30% | Define, list, recall key facts |
| Understand | 40% | Explain, describe, summarize |
| Apply | 20% | Use a concept in a given context |
| Analyze | 10% | Compare, distinguish, relate |

### Scoring Criteria

- **Correct (1.0)**: The response includes the key facts or terms and demonstrates understanding at the targeted Bloom level. The core concept must be correct.
- **Incorrect (0.0)**: The response is missing, wrong, incomplete to the point of missing the core concept, or demonstrates a fundamental misconception.

Partial credit is intentionally excluded. Study questions test whether the learner has the foundational knowledge needed for deeper work. A partially correct answer on a factual question signals a gap that must be addressed before proceeding to mastery-level material.

### Count Per Module

Each module generates 15-20 study questions during the `/enrich` workflow.

---

## 3. Mastery Question Scoring

Mastery questions require written analysis and are scored on a **5-point rubric across 5 dimensions**, yielding a composite score from 0.00 to 1.00.

### Bloom's Level Distribution

| Bloom Level | Proportion | Example |
|-------------|-----------|---------|
| Apply | 25% | Use a framework in a new scenario |
| Analyze | 30% | Break down a system, identify tradeoffs |
| Evaluate | 25% | Assess, critique, justify a decision |
| Create | 20% | Design, propose, synthesize a new approach |

### Scoring Scale

| Score | Normalized | Label | Criteria |
|-------|-----------|-------|----------|
| 5 | 1.0 | Excellent | Covers all rubric points, demonstrates original thinking, connects to broader context |
| 4 | 0.8 | Strong | Covers most rubric points, clear reasoning, minor gaps |
| 3 | 0.6 | Adequate | Covers core rubric points, some reasoning gaps, surface-level connections |
| 2 | 0.4 | Developing | Missing key rubric points, reasoning is unclear or incomplete |
| 1 | 0.2 | Beginning | Significant gaps, demonstrates fundamental misunderstanding |
| 0 | 0.0 | No attempt | No response or completely off-topic |

### Evaluation Dimensions

Each mastery response is evaluated across five dimensions, each scored 0-5:

| Dimension | What It Measures |
|-----------|-----------------|
| **Accuracy** | Factual correctness of claims and concepts |
| **Depth** | Goes beyond surface-level; explains mechanisms, not just facts |
| **Connections** | Links to related concepts, modules, or real-world applications |
| **Examples** | Provides relevant, original examples that demonstrate understanding |
| **Clarity** | Logical structure, clear expression, no ambiguity |

### Composite Calculation

```
mastery_score = (accuracy + depth + connections + examples + clarity) / 5 / 5
```

Each dimension is scored 0-5. The average of the five dimensions is divided by 5 to normalize the result to a 0.00-1.00 range.

### Sample Question and Scored Responses

**Question** (Bloom level: Evaluate)

> You are designing an AI-powered product recommendation engine for an e-commerce platform. The team proposes using a deep learning model trained on purchase history. Evaluate this approach: what are its strengths, what risks does it introduce, and what alternative or complementary approaches would you recommend? Justify your reasoning.

---

**Score 1 (Beginning, 0.2)** -- Learner response:

> "Deep learning is good for recommendations because it can learn patterns. It might have some risks like bias. You could also use collaborative filtering."

Evaluation: The response names a strength and an alternative but at a surface level only. No mechanisms are explained, no specific risks are identified, no examples are given, and the justification is absent. Accuracy: 1, Depth: 1, Connections: 1, Examples: 0, Clarity: 2. Composite: 5 / 25 = 0.20.

---

**Score 3 (Adequate, 0.6)** -- Learner response:

> "A deep learning model trained on purchase history can identify complex, non-linear patterns that simpler models miss, such as cross-category preferences. However, it risks overfitting to existing purchase data, which means it may not surface novel products. There is also a cold-start problem for new users who have no history. Collaborative filtering or content-based filtering could complement the approach. I would recommend a hybrid system that uses deep learning for users with rich history and content-based methods for new users."

Evaluation: Core strengths and risks are identified with some mechanism explanation. The hybrid recommendation shows application-level thinking. However, the response does not address fairness/bias risks, does not connect to broader product design principles, and provides no concrete examples. Accuracy: 4, Depth: 3, Connections: 2, Examples: 2, Clarity: 4. Composite: 15 / 25 = 0.60.

---

**Score 5 (Excellent, 1.0)** -- Learner response:

> "A deep learning recommendation engine trained on purchase history offers several strengths: it can capture complex, non-linear relationships between products and users (e.g., recognizing that customers who buy running shoes in spring also tend to buy sunscreen); it scales well with large datasets; and it can surface latent features that rule-based systems miss.
>
> However, this approach introduces significant risks. First, feedback loop bias: the model recommends what sells, reinforcing existing patterns rather than surfacing genuinely useful products, which can create a filter bubble. Second, the cold-start problem means new users and new products have no signal, leading to poor initial experiences. Third, deep learning models are opaque, making it difficult to explain recommendations to users or debug unexpected behavior, which matters for user trust and regulatory compliance (e.g., the EU AI Act's transparency requirements).
>
> I would recommend a hybrid architecture: use deep learning as one signal alongside collaborative filtering (for social proof patterns) and content-based filtering (for cold-start mitigation). Additionally, incorporate an exploration mechanism, such as epsilon-greedy or Thompson sampling, to intentionally surface novel products and counteract feedback loops. Finally, build an explanation layer that can provide human-readable justifications, since research from the Responsible AI literature shows that explainable recommendations increase user trust and conversion rates by 10-15%."

Evaluation: Comprehensive analysis covering strengths, multiple specific risks with mechanisms explained, connections to regulatory context and responsible AI research, concrete examples throughout, and a well-structured recommendation with justification. Accuracy: 5, Depth: 5, Connections: 5, Examples: 5, Clarity: 5. Composite: 25 / 25 = 1.00.

### Count Per Module

Each module generates 8-12 mastery questions during the `/enrich` workflow, with difficulty progressing from easier to harder.

---

## 4. Tutor Session Scoring

Tutor sessions (invoked via `/tutor` or `/study` in Socratic mode) are scored across **5 dimensions**, each on a 0-100 scale. These dimensions align with Bloom's taxonomy, progressing from basic recall through synthesis and teaching.

### Dimensions and Level Descriptions

#### Recall (0-100)

Can retrieve relevant facts.

| Band | Score Range | Description |
|------|------------|-------------|
| Beginning | 0-25 | Cannot recall basic facts even with prompting |
| Developing | 26-50 | Recalls some facts but with significant gaps or errors |
| Proficient | 51-75 | Recalls most facts correctly with minor gaps |
| Mastery | 76-100 | Recalls all relevant facts quickly and accurately |

#### Understanding (0-100)

Explains mechanisms, not just terms.

| Band | Score Range | Description |
|------|------------|-------------|
| Beginning | 0-25 | Can only repeat definitions without explaining how or why |
| Developing | 26-50 | Partial explanation; mixes up cause and effect or key mechanisms |
| Proficient | 51-75 | Explains core mechanisms correctly; some nuance missing |
| Mastery | 76-100 | Explains mechanisms fully with nuance and appropriate caveats |

#### Application (0-100)

Uses concept in new context.

| Band | Score Range | Description |
|------|------------|-------------|
| Beginning | 0-25 | Cannot apply concept outside the exact context presented |
| Developing | 26-50 | Applies concept in obvious cases but struggles with novel contexts |
| Proficient | 51-75 | Applies concept to new contexts with minor errors or gaps |
| Mastery | 76-100 | Fluently applies concept to novel and complex contexts |

#### Integration (0-100)

Connects to other concepts and modules.

| Band | Score Range | Description |
|------|------------|-------------|
| Beginning | 0-25 | Treats concept in isolation; no connections made |
| Developing | 26-50 | Makes superficial connections when prompted |
| Proficient | 51-75 | Identifies meaningful connections across topics or modules |
| Mastery | 76-100 | Spontaneously synthesizes across modules with original insight |

#### Teaching (0-100)

Explains clearly to others (teach-back technique).

| Band | Score Range | Description |
|------|------------|-------------|
| Beginning | 0-25 | Explanation is confusing or inaccurate |
| Developing | 26-50 | Explanation covers basics but is disorganized or incomplete |
| Proficient | 51-75 | Clear explanation with good structure; minor gaps |
| Mastery | 76-100 | Excellent teach-back: clear, accurate, well-structured, uses examples |

### Composite Calculation

```
tutor_score = (recall + understanding + application + integration + teaching) / 5 / 100
```

The average of the five dimensions (each 0-100) is divided by 100, yielding a normalized score from 0.00 to 1.00.

### Interaction-Level Tracking

Each interaction within a tutor session is tagged with:

- **Technique**: elaborative-interrogation, teach-back, concrete-examples, interleaving, or socratic-chain
- **Dimension scores**: Which dimensions this interaction tested
- **Meta-feedback**: Quality rating (strong / developing / emerging), rationale, and a technique recommendation for improvement

At the session level, the system tracks reasoning patterns, effective techniques, and growth areas.

---

## 5. Module Composite Score

The module composite score combines all three assessment types into a single 0.00-1.00 value:

```
module_score = (study_questions * 0.20) + (mastery_questions * 0.50) + (tutor_sessions * 0.30)
```

### Component Breakdown

| Component | Weight | Score Source | Range |
|-----------|--------|-------------|-------|
| `study_questions` | 0.20 | Average of binary study question scores (correct count / total count) | 0.00 - 1.00 |
| `mastery_questions` | 0.50 | Average composite of all mastery question rubric evaluations | 0.00 - 1.00 |
| `tutor_sessions` | 0.30 | Average composite of all tutor session dimension scores | 0.00 - 1.00 |

### Why These Weights

- **Study questions at 20%** ensure the learner has built a factual foundation but prevent high scores from recall alone. A learner who aces study questions but struggles with mastery and tutor work will score at most 0.20, which is well below passing.
- **Mastery questions at 50%** are the primary driver of the score because they require analysis, evaluation, and synthesis. These are the competencies that distinguish surface familiarity from genuine understanding.
- **Tutor sessions at 30%** capture real-time reasoning that written answers cannot. The Socratic method exposes whether a learner can think through problems, handle follow-up questions, and teach concepts back, which are skills that demonstrate transferable mastery.

### Example Calculation

A learner has the following scores for Module 3:

- Study questions: 0.85 (17/20 correct)
- Mastery questions: 0.72 (average rubric composite)
- Tutor sessions: 0.68 (average dimension composite)

```
module_score = (0.85 * 0.20) + (0.72 * 0.50) + (0.68 * 0.30)
             = 0.170 + 0.360 + 0.204
             = 0.734  (Passing)
```

---

## 6. Status Levels and Fluency Threshold

Each module (and the cumulative course score) maps to a status level:

| Status | Score Range | Meaning |
|--------|------------|---------|
| Below Passing | < 0.70 | Needs significant review. Core concepts not yet solid. |
| Passing | 0.70 - 0.79 | Meets minimum threshold. Knows the basics but has gaps. |
| Proficient | 0.80 - 0.89 | Strong understanding. Can reliably apply concepts. |
| Mastery | >= 0.90 | Deep, transferable knowledge. Can teach and synthesize. |

### Fluency Threshold: 0.80 (Proficient)

A learner must reach **Proficient (0.80)** to advance to the next module. This threshold is intentionally set above Passing (0.70) because:

- **Passing (0.70)** indicates the learner knows the basics but has identifiable gaps. Advancing with gaps creates a compounding problem as later modules build on earlier ones.
- **Proficient (0.80)** indicates the learner can reliably apply concepts in new contexts and has addressed major gaps. This level provides a stable foundation for more advanced material.
- **Mastery (0.90)** is aspirational. Requiring it for advancement would slow progress without proportional benefit, since the SRS system continues to reinforce material after advancement.

---

## 7. Bloom's Taxonomy Integration

All questions are tagged with their targeted Bloom level. The system tracks per-module Bloom breakdowns to reveal **where** a learner excels versus struggles, not just their overall score.

### Distribution by Question Type

**Study questions** emphasize the lower Bloom levels:

| Bloom Level | Proportion |
|-------------|-----------|
| Remember | 30% |
| Understand | 40% |
| Apply | 20% |
| Analyze | 10% |

**Mastery questions** target the higher Bloom levels:

| Bloom Level | Proportion |
|-------------|-----------|
| Apply | 25% |
| Analyze | 30% |
| Evaluate | 25% |
| Create | 20% |

### Bloom Breakdown in Score Reports

The score report tracks performance at each Bloom level per module:

```yaml
bloom_breakdown:
  remember: 0.90    # Strong recall
  understand: 0.85  # Good comprehension
  apply: 0.72       # Can apply but with gaps
  analyze: 0.65     # Struggles to break down problems
  evaluate: 0.58    # Difficulty making judgments
  create: 0.50      # Needs work on synthesis
```

This breakdown enables targeted recommendations. In the example above, the learner recalls and comprehends well but struggles with higher-order thinking. The system would recommend more mastery questions focused on Analyze and Evaluate, and tutor sessions emphasizing the teach-back technique to strengthen synthesis skills.

---

## 8. Trend Calculation

Trends measure whether a learner is improving, stable, or declining over recent sessions.

### Parameters

- **Window size**: 3 sessions (the most recent 3 scored sessions for a module)
- **Threshold**: 5% (0.05)

### Classification

| Trend | Condition | Signal |
|-------|-----------|--------|
| Improving | Score increased >= 5% over the 3-session window | Learner is making progress; continue current approach |
| Stable | Score changed within +/- 5% | Learner may be plateauing; consider varying techniques |
| Declining | Score decreased >= 5% | Learner is struggling; review approach and address gaps |

The trend is calculated by comparing the most recent session score to the earliest session in the window. A declining trend triggers a recommendation to revisit the module's weakest topics before continuing.

---

## 9. SRS Retention Rate

The spaced repetition system (SRS) uses a modified SM2 algorithm. Retention rate is a cumulative metric that feeds into the overall scorecard.

### Definition

```
srs_retention_rate = (items with quality >= 3 on most recent review) / (total active items)
```

On the SM2 quality scale:

| Quality | Meaning |
|---------|---------|
| 0 | Complete blackout -- no memory at all |
| 1 | Incorrect -- but recognized the answer when shown |
| 2 | Incorrect -- but the answer felt familiar |
| 3 | Correct -- but required significant effort to recall |
| 4 | Correct -- with some hesitation |
| 5 | Perfect -- instant recall with confidence |

A quality of 3 or higher counts as "recalled correctly" for the retention rate calculation. Items scored below 3 are reset (interval returns to 1 day, repetitions return to 0).

### Targets

- **Target retention rate**: 85%+ of items at quality >= 3
- **Warning threshold**: Below 70% triggers a recommendation for dedicated review sessions
- **Session cap**: When the SRS queue exceeds 30 due items, reviews are capped at 20 per session to prevent fatigue

### Integration with Scoring

The SRS retention rate appears in the cumulative section of the score report alongside the total active items count. While it does not directly factor into the module composite formula, a declining retention rate signals that foundational knowledge is eroding and surfaces as a priority recommendation.

---

## Feedback Guidelines

All scoring produces specific, actionable feedback:

- Reference what was strong and what was missing
- For incorrect answers, explain the correct answer and why
- For partial answers, acknowledge what was correct before addressing gaps
- Reference rubric criteria when explaining scores
- Adapt feedback tone to the learner's preference (encouraging, blunt, or balanced)

Scoring is executed at **temperature 0.3** (highly deterministic) to ensure consistency across evaluations. The system never inflates scores and always references specific evidence from the learner's response.

### Exceptions

- During initial orientation (first session on a module), scoring is relaxed to focus on engagement rather than accuracy
- Self-rated SRS quality scores (0-5) are accepted at face value without tutor evaluation
