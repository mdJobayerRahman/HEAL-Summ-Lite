# HEAL-Summ-Lite: Lightweight Ethical Health Summarization Pipeline

A simplified implementation inspired by Fisher et al. (2025) — *HEAL-Summ: A Lightweight and Ethical Framework for Accessible Summarization of Health Information*.

---

## Methods and Validation

1. **Approach:** Used Phi-3 Mini (3.8B) via Ollama for local CPU-based health text summarization with a **two-shot prompting strategy** — two simplified examples (~160 words each) teach the model to produce readable, full-coverage summaries. The pipeline has 5 stages: LLM summarization → post-processing → readability scoring → 10 heuristic checks → tiered human review.

2. **Why two-shot prompting:** Initial zero-shot attempts produced summaries that only paraphrased the first 3-4 paragraphs sequentially and never reached later sections (prevention, treatment, recommendations). Two-shot examples taught the model through in-context learning to cover beginning, middle, AND end sections.

3. **Assumption — sequential bias:** Phi-3 Mini generates tokens left-to-right with no planning ability. It cannot "read the whole text and decide what matters most" like larger models. The two-shot examples compensate by demonstrating the expected full-coverage pattern, and simplified example language (FKGL ~5) guides the model toward plain-language output.

4. **Assumption — post-processing is acceptable:** The model consistently omitted safety caveats and source attributions, so post-processing automatically adds "Talk to a doctor before making health choices" and source references (WHO/CDC/NHS) when missing. This is transparent and documented — not hidden from the reviewer.

5. **What worked:** Two-shot prompting successfully produced complete summaries within the 120–180 word target range across all 5 texts. Simplified examples (8–12 words per sentence) brought FKGL scores down from 12–18 (zero-shot) to 6.9–10.1. The 10 heuristic checks provided instant, reproducible quality flags, and the tiered human review system (CRITICAL/ESCALATE/REVIEW/APPROVED) gave clear escalation decisions.

6. **Failure case — TextRank and TF-IDF:** Before two-shot prompting, I tried TextRank (graph-based) and TF-IDF (term frequency) extraction to pre-select key sentences. Both failed — removing sentences before the LLM saw them caused ~50% entity loss, lower readability, and insufficient word count. The LLM needs the full text to preserve all statistics and entities.

7. **Failure case — readability vs. information:** There is a fundamental tension between simplifying language (lowering FKGL) and preserving medical terms and statistics. The semantic similarity heuristic flagged most summaries at 1–4% similarity (possible hallucination), but this was a false positive — simplified wording naturally diverges from complex source text. Numeric coverage ranged from 0–20% because the model drops specific numbers when simplifying.

8. **Evaluation method:** 10 rule-based heuristic checks run instantly on every summary — covering numeric coverage, entity coverage, compression ratio, safety caveat presence, negation flip detection, hedging preservation, source attribution, sentence count, semantic similarity, and sensitive topic flagging. Readability is measured via Flesch-Kincaid Grade Level (target ≤12) and Flesch Reading Ease (target ≥30).

9. **What I would improve with more time:** Implement LLM-as-judge evaluation for nuanced quality assessment, use semantic embedding similarity (MiniLM) instead of string matching to reduce false positives, add proper Named Entity Recognition via spaCy instead of regex, and test a multi-model ensemble for hallucination detection as described in the original HEAL-Summ paper.
=======
- **LLM-based evaluation system** to replace rule-based heuristic risk flags with more nuanced assessment
- **Time optimization for summary generation** — current ~5 min/summary on CPU is too slow for production use
- **Multi-model ensemble** for hallucination detection via inter-model agreement (as in original HEAL-Summ)
- **Better emotion and context integration** — started but limited by project timeline
- **Proper Named Entity Recognition (NER)** using spaCy instead of regex patterns
- **Toxicity detection** using RoBERTa-based classifier

---

## Results

| ID | Title | Original Words | Summary Words | FKGL | FRE | Human Review Decision |
|----|-------|---------------|---------------|------|-----|----------------------|
| TEXT_01 | Diabetes – Key Facts a.. | 605 | 158 | 9.4 | 62.6 | ESCALATE |
| TEXT_02 | About Influenza (Flu) .. | 607 | 152 | 7.0 | 69.2 | ESCALATE |
| TEXT_03 | High Blood Pressure (H.. | 638 | 173 | 10.0 | 62.1 | ESCALATE |
| TEXT_04 | Malaria – Key Facts, P.. | 657 | 153 | 10.1 | 56.9 | ESCALATE |
| TEXT_05 | Mental Health – Unders.. | 693 | 164 | 6.9 | 75.2 | CRITICAL |

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
>>>>>>> e55441302d2c2661efd45942341ea999273f804d

---

## Tool Use Disclosure

- **Google AI Studio (Gemini):** Used as a strategic brainstorming partner to design the clinical risk heuristics (e.g., identifying Directionality Flips and Disparity Erasure) and to structure the logic for the multi-tiered human review protocol.
- **Claude (Anthropic):** Utilized for some code generation, structuring the Python evaluation pipeline, formatting the pandas outputs, and debugging the heuristic logic loops.
- **ChatGPT 5.2:** Used strictly for benchmark comparison to evaluate how a frontier model handles full-context health summarization versus the outputs of the smaller, locally deployed Phi-3 model.

---

## References

- Fisher, A. et al. (2025). *HEAL-Summ: A Lightweight and Ethical Framework for Accessible Summarization of Health Information*
- GitHub: [andrfish/FiM-Lightweight-LLM-Summarization-Framework](https://github.com/andrfish/FiM-Lightweight-LLM-Summarization-Framework)
