# HEAL-Summ-Lite: Lightweight Ethical Health Summarization Pipeline

A simplified implementation inspired by Fisher et al. (2025) — *HEAL-Summ: A Lightweight and Ethical Framework for Accessible Summarization of Health Information*.

---

## Architecture Overview

```
Original Health Text (600-700 words)
    │
    ▼
[Stage 1] Two-Shot Phi-3 Mini Summarization (1 LLM call, ~2-5 min)
    │
    ▼
[Stage 2] Post-Processing (auto-add caveat, source, break long sentences)
    │
    ▼
[Stage 3] Readability Scoring (FKGL + FRE)
    │
    ▼
[Stage 4] 10 Rule-Based Heuristic Checks
    │
    ▼
[Stage 5] Tiered Human Review Decision
    │
    ▼
Results Tables + Severity Report
```

---

## Project Summary & Methodology

### Approach Used
A **two-shot prompting strategy** was implemented to teach the model specific summarization patterns. Standard zero-shot attempts resulted in incomplete summaries that only covered the beginning of articles, missing critical information from later sections.

### Why Two-Shot Prompting?
Initial attempts using **TextRank and TF-IDF** pre-extraction caused significant loss of entities—often dropping nearly half of the required names or numbers. Two-shot prompting improved:
- Context retention (covers beginning, middle, AND end of articles)
- Readability scores (model mimics the simple style of examples)
- Safety caveat inclusion (examples demonstrate proper disclaimer placement)

### Assumptions Made
- The smaller model's sequential "left-to-right" processing style was the primary cause for dropping content at the end of long contexts once token limits were approached
- Simplified two-shot examples (FKGL ~5) would guide the model to produce simpler output
- Post-processing fixes (auto-adding caveats and sources) would address consistent model omissions
- 120-180 word summaries are appropriate length for public health communication

### What Worked
- **Two-shot prompting** successfully trained the model to provide "complete" summaries with proper conclusions
- **Simplified examples** (8-12 words per sentence) improved readability compared to complex examples
- **Post-processing** reliably added missing safety caveats and source attributions
- **10 heuristic checks** provided instant, reproducible, auditable quality assessment
- **Tiered human review** gave clear escalation criteria (CRITICAL → ESCALATE → REVIEW → APPROVED)

### What Didn't Work (Failure Cases)
- **TF-IDF and TextRank pre-extraction** failed because they significantly increased missing entities (e.g., eliminating 7 out of 15 key points)
- **Readability targets were difficult to meet** — Phi-3 Mini consistently produced FKGL scores of 12-18 despite prompts requesting 8th-grade level writing
- **Semantic similarity heuristic** flagged many summaries as "possible hallucination" (3-7% similarity) because simplified wording differs significantly from source text — this was a false positive in most cases
- **Word count constraints** were frequently ignored by the model despite explicit instructions

### Evaluation Method
- **Primary check:** Manual human review for escalated cases to verify factual and structural soundness
- **Readability metrics:** Flesch-Kincaid Grade Level (FKGL) and Flesch Reading Ease (FRE)
  - Target: FKGL ≤ 12, FRE ≥ 30
  - Observation: FKGL improved with simplified prompting, but FRE tended to degrade for longer, more complex health topics
- **Heuristic flags:** 10 automated checks for numeric coverage, entity coverage, caveat presence, negation flips, etc.

---

## 10 Heuristic Quality Checks

| # | Check | What It Catches | Severity |
|---|-------|-----------------|----------|
| 1 | Numeric Coverage | Missing statistics/percentages | WARNING |
| 2 | Entity Coverage | Missing drug/organization names | WARNING |
| 3 | Compression Ratio | Summary too long or too short | WARNING |
| 4 | Caveat Presence | Treatment mentioned without safety disclaimer | CRITICAL |
| 5 | Negation Flip | "does NOT spread" → "spreads" | CRITICAL |
| 6 | Hedging Preservation | "may cause" → "causes" (false certainty) | WARNING |
| 7 | Source Attribution | Missing WHO/CDC/NHS mention | INFO |
| 8 | Sentence Count | Too few or too many sentences | WARNING |
| 9 | Semantic Similarity | Possible hallucination or copy-paste | WARNING |
| 10 | Sensitive Topics | Crisis resources missing for mental health content | CRITICAL |

---

## Human Review Escalation Rules

| Tier | Condition | Action |
|------|-----------|--------|
| **CRITICAL** | Any CRITICAL flag triggered | Always escalate to human reviewer |
| **ESCALATE** | 2+ WARNING flags | Escalate to human reviewer |
| **REVIEW** | 1 WARNING flag | Soft flag for optional review |
| **APPROVED** | No flags | Safe to publish |

---

## Limitations & Resource Constraints

Due to resource constraints (CPU-only inference, ~5 minutes per summary), the following limitations apply:

1. **No LLM-based evaluation** — Could not implement LLM-as-judge for quality assessment due to high latency and computational cost
2. **No multi-model comparison** — Unlike the original HEAL-Summ paper which uses 3 models for hallucination detection, this implementation uses only Phi-3 Mini
3. **No semantic embedding similarity** — Could not use MiniLM or similar models for proper semantic similarity scoring
4. **Limited prompt iteration** — Resource constraints limited ability to test many prompt variations
5. **Readability targets not fully met** — Model consistently produces FKGL 12-18 despite targeting <10

---

## Future Improvements

Given more time and resources, the following improvements would be implemented:

- **LLM-based evaluation system** to replace rule-based heuristic risk flags with more nuanced assessment
- **Time optimization for summary generation** — current ~5 min/summary on CPU is too slow for production use
- **Multi-model ensemble** for hallucination detection via inter-model agreement (as in original HEAL-Summ)
- **Better emotion and context integration** — started but limited by project timeline
- **Proper Named Entity Recognition (NER)** using spaCy instead of regex patterns
- **Toxicity detection** using RoBERTa-based classifier

---

## Setup & Installation

### Prerequisites
- Python 3.10+
- Ollama installed and running
- 8GB+ RAM recommended

### Installation

```bash
# 1. Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# 2. Pull the model
ollama pull phi3:mini

# 3. Start Ollama server
ollama serve

# 4. Install Python dependencies
pip install requests textstat tabulate

# 5. Run the notebook
jupyter notebook HEAL_Summ_Lite_updated.ipynb
```

---

## File Structure

```
heal-summ-lite/
├── HEAL_Summ_Lite_updated.ipynb   # Main pipeline notebook
├── health_texts.py                 # 5 sample health articles
├── heal_summ_results.json          # Output results (generated)
└── README.md                       # This file
```

---

## Tool Use Disclosure

### Tools Used

| Tool | Purpose |
|------|---------|
| **Ollama + Phi-3 Mini** | Local LLM for health article summarization |
| **Google AI Studio** | Primary model interaction and testing |
| **Claude (Anthropic)** | Model assessment, code generation and organization, debugging |
| **ChatGPT** | Benchmark comparison — how larger models handle full-context summaries vs. smaller sequential models |
| **Jupyter Notebooks** | Running initial TF-IDF and TextRank experiments |
| **textstat** | Readability score calculation (FKGL, FRE) |
| **Python regex** | Entity and number extraction |

### What I Personally Did
- Designed the overall pipeline architecture
- Selected and tested the two-shot prompting approach after TF-IDF/TextRank failed
- Reviewed and validated all code before implementation
- Ran the complete pipeline and analyzed results
- Made decisions about threshold values (FKGL > 12, number coverage < 30%, etc.)
- Wrote the heuristic check logic based on HEAL-Summ paper methodology
- Identified failure cases and documented limitations
- Validated that post-processing fixes correctly addressed missing caveats/sources

### What I Learned
- How lightweight LLMs (3B parameters) can be used for health text summarization
- The trade-off between readability and information preservation
- Why two-shot prompting outperforms extractive methods (TF-IDF/TextRank) for this task
- How to design rule-based quality checks for LLM outputs
- The importance of human review rules in safety-critical health domains

---

## References

- Fisher, A. et al. (2025). *HEAL-Summ: A Lightweight and Ethical Framework for Accessible Summarization of Health Information*
- GitHub: [andrfish/FiM-Lightweight-LLM-Summarization-Framework](https://github.com/andrfish/FiM-Lightweight-LLM-Summarization-Framework)

---

## Sample Output

```
======================================================================
  Processing: TEXT_04 — Malaria – Key Facts, Prevention, and Treatment
  Source: World Health Organization (WHO)
  Original: 657 words
======================================================================

  [1/5] Two-shot summarization (Phi-3 Mini, temp=0.3)...
        ✓ Raw summary: 160 words in 285.3s
  [2/5] Applying post-processing fixes...
        ✓ Added safety caveat
        ✓ Added source attribution
        ✓ Post-processed: 175 words
  [3/5] Computing readability metrics...
        ⚠ FKGL: 13.2 (target: ≤12)
        ✓ FRE:  42.5 (target: ≥30)
  [4/5] Running 10 rule-based heuristic checks...
        Results: 7 PASS, 3 FLAG
  [5/5] Applying tiered human review rules...
        Decision: ESCALATE
```

---

*Last updated: February 2025*
