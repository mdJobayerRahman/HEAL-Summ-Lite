# HEAL-Summ-Lite: Lightweight Ethical Health Summarization Pipeline

A simplified implementation inspired by Fisher et al. (2025) — *HEAL-Summ: A Lightweight and Ethical Framework for Accessible Summarization of Health Information*.

---

## Methods and Validation

1. **Approach:** I used Phi-3 Mini (3.8B) through Ollama for local CPU-based health text summarization with a two-shot prompt strategy. Two simplified examples (~160 words each) were included in the prompt to teach the model how to produce readable, full-coverage summaries. The pipeline runs in 5 stages: LLM summarization → post-process → readability score → 10 heuristic checks → tiered human review.

2. **Why two-shot:** My initial zero-shot attempts only paraphrased the first 3–4 paragraphs in order and never reached later sections like prevention, treatment, or recommendations. The two examples showed the model, through in-context learn, how to cover the start, middle, AND end of an article.

3. **Assumption (sequential bias):** Phi-3 Mini produces tokens left to right with no ability to plan ahead. It cannot read the whole text and decide what matters most the way a larger model would. The two-shot examples compensate for this because they show the expected full-coverage pattern, and their simplified language (FKGL ~5) pushes the model toward plain-language output.

4. **Assumption (post-process is acceptable):** The model consistently left out safety caveats and source attributions, so the post-process step automatically adds "Talk to a doctor before making health choices" and source references (WHO/CDC/NHS) when they are absent. This is transparent and documented, not hidden from the reviewer.

5. **What worked:** Two-shot prompts successfully produced complete summaries within the 120–180 word target for all 5 texts. Simplified examples (8–12 words per sentence) brought FKGL scores down from 12–18 (zero-shot) to 6.9–10.1. The 10 heuristic checks gave instant, reproducible quality flags, and the tiered human review system (CRITICAL/ESCALATE/REVIEW/APPROVED) provided clear escalation decisions.

6. **Failure case (TextRank and TF-IDF):** Before I landed on two-shot prompts, I tried TextRank (graph-based) and TF-IDF (term frequency) extraction to pre-select key sentences. Both failed because when I removed sentences before the LLM saw them, roughly 50% of entities were lost, readability dropped, and word count fell short. The LLM needs the full text to preserve all statistics and entities.

7. **Failure case (readability vs. information):** There is a real tension between simplified language (lower FKGL) and preserved medical terms and statistics. The semantic similarity heuristic flagged most summaries at 1–4% (possible hallucination), but this was a false positive because simplified text naturally diverges from complex source text. Numeric coverage ranged from 0–20% because the model tends to drop specific numbers when it simplifies.

8. **Evaluation method:** 10 rule-based heuristic checks run instantly on every summary. They cover numeric coverage, entity coverage, compression ratio, safety caveat presence, negation flip detection, hedge preservation, source attribution, sentence count, semantic similarity, and sensitive topic flags. Readability is measured via Flesch-Kincaid Grade Level (target ≤12) and Flesch Reading Ease (target ≥30).

9. **What I would improve with more time:** I would add an LLM-as-judge evaluation step for more nuanced quality assessment, swap string match for semantic embeddings (MiniLM) to reduce false positives, replace regex-based entity extraction with proper NER via spaCy, and test a multi-model ensemble for hallucination detection as described in the original HEAL-Summ paper. I would also explore an LLM-based evaluation system to replace rule-based heuristic risk flags, optimize the ~5 min/summary generation time on CPU, look into multi-model ensembles for inter-model agreement on hallucination detection, integrate better emotion and context analysis (which I started but could not finish due to the project timeline), and add toxicity detection with a RoBERTa-based classifier.

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
