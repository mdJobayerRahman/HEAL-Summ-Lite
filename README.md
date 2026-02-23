# HEAL-Summ-Lite

**A Lightweight and Ethical Framework for Accessible Summarization of Health Information**

Inspired by: *HEAL-Summ: a Lightweight and Ethical Framework for Accessible Summarization of Health Information*

---

## Architecture (Hybrid: Rule-Based + LLM-Based)

```
Original Health Text
        │
        ▼
┌──────────────────────────────────────────┐
│  STAGE 1: Phi-3 Mini 3.8B (temp=0.3)    │
│  PROMPT 1 — Summarization                │
│  • 120-180 word target                   │
│  • Preserve all stats and entities       │
│  • Plain language (grade 8)              │
│  • Add treatment caveats                 │
└──────────────────────────────────────────┘
        │ Summary
        ▼
┌──────────────────────────────────────────┐
│  STAGE 2: Python (Deterministic)         │
│  • Word count verification               │
│  • FKGL (grade level)                    │
│  • FRE (reading ease)                    │
└──────────────────────────────────────────┘
        │ Readability scores
        ▼
┌──────────────────────────────────────────┐
│  STAGE 3: Python (Rule-Based Heuristics) │  ← LAYER 1: Deterministic
│  • Numeric coverage (% of numbers kept)  │
│  • Entity coverage (% of names kept)     │
│  • Compression ratio check               │
│  • Caveat keyword detection              │
└──────────────────────────────────────────┘
        │ Rule-based flags
        ▼
┌──────────────────────────────────────────┐
│  STAGE 4: Phi-3 Mini 3.8B (temp=0.0)    │  ← LAYER 2: Contextual
│  PROMPT 2 — Quality Evaluation           │
│  INPUT: Original + Summary (both)        │
│  • Factual fidelity (semantic)           │
│  • Numeric accuracy (not just presence)  │
│  • Entity completeness (contextual)      │
│  • Caveat appropriateness (contextual)   │
│  OUTPUT: Structured JSON flags           │
└──────────────────────────────────────────┘
        │ LLM flags
        ▼
┌──────────────────────────────────────────┐
│  STAGE 5: Python (Combined Decision)     │
│  Escalate if:                            │
│  • ANY rule-based flag = FLAG            │
│  • ANY LLM flag = FLAG                   │
│  • FKGL > grade 10                       │
│  • FRE < 30                              │
│  • Word count outside 120-180            │
└──────────────────────────────────────────┘
        │
        ▼
    Results Table (3 tables: Overview, LLM Checks, Rule-Based Checks)
```

---

## Why Hybrid? (Key Design Decision)

This system uses **two layers** of quality checking, each with different strengths:

| Aspect | Rule-Based (Layer 1) | LLM-Based (Layer 2) |
|--------|---------------------|---------------------|
| **Speed** | Instant | 15-30 seconds |
| **Determinism** | 100% reproducible | May vary slightly |
| **What it catches** | Missing numbers/entities (surface) | Flipped statistics, inappropriate claims (semantic) |
| **Failure mode** | Misses paraphrased equivalents | Self-evaluation bias |
| **Auditability** | Fully traceable | Requires reading model reasoning |

**Together:** Rule-based catches what the LLM might self-approve. LLM catches what rules cannot reason about. This is standard practice in production NLP pipelines.

---

## Setup Instructions

### Step 1: Install Ollama
Download from: https://ollama.com/download

### Step 2: Pull Phi-3 Mini
```bash
ollama pull phi3:mini
```

### Step 3: Ensure Ollama is Running
```bash
ollama serve
```

### Step 4: Install Python Dependencies
```bash
pip install -r requirements.txt
```

### Step 5: Run the Pipeline
```bash
python heal_summ_lite.py
```
Expected runtime: 3-10 minutes (5 texts x 2 LLM calls each).

---

## File Structure

```
heal_summ_lite/
├── heal_summ_lite.py      ← Main pipeline (prompts, heuristics, evaluation, tables)
├── health_texts.py        ← 5 public health texts with source attribution
├── requirements.txt       ← Python dependencies
├── README.md              ← This file
└── heal_summ_results.json ← Generated after running (full results in JSON)
```

---

## Quality Check Dimensions

### Layer 1: Rule-Based Heuristics (Python)

| Check | Method | Threshold |
|-------|--------|-----------|
| **Numeric Coverage** | Extract all numbers from original and summary via regex, compute overlap % | FLAG if < 50% preserved |
| **Entity Coverage** | Extract org names, drug names, diseases via pattern matching, compute overlap % | FLAG if < 40% preserved |
| **Compression Ratio** | summary_words / original_words | FLAG if < 0.15 or > 0.85 |
| **Caveat Check** | If treatment keywords in original, check for safety phrases in summary | FLAG if treatment content without caveat |

### Layer 2: LLM-Based Evaluation (Phi-3 Mini)

| Check | What It Reasons About |
|-------|----------------------|
| **Factual Fidelity** | Does the summary contain ANY claim not supported by the original? |
| **Numeric Preservation** | Are numbers not just present but ACCURATE? (catches "40% reduction" to "40% increase") |
| **Entity Completeness** | Are important entities preserved in context? (not just string matching) |
| **Caveat Presence** | Is safety language contextually appropriate for the content? |

---

## Human Review Policy

**When to escalate (any ONE of these triggers human review):**
- Any LLM quality dimension flagged
- Any rule-based heuristic flagged
- Readability above grade 10 (FKGL) or below 30 (FRE)
- Summary word count outside 120-180 range
- Evaluation JSON could not be parsed

**Philosophy:** This system is conservative by design. For health content, false positives (unnecessary human review) are far less costly than false negatives (missed errors that could harm someone). The system is a **first-pass filter**, not a replacement for expert review.

---

## Known Limitations

1. **Self-evaluation bias:** The same model generates and evaluates summaries. Mitigated by forcing side-by-side comparison in Prompt 2, but not eliminated.
2. **3.8B model size:** May miss subtle errors. Mitigated by conservative escalation (any flag triggers human review) and supplemental rule-based checks.
3. **JSON output reliability:** Small models produce inconsistent JSON. Mitigated by a 4-strategy fallback parser; parse failure triggers review.
4. **Simple entity extraction:** Uses regex patterns, not a full NER model. Mitigated by LLM entity evaluation in Layer 2.
5. **Not clinically validated:** This is a screening tool, not a replacement for health communication professionals.

---

## Tools and Transparency

| Component | Tool | Why This Choice |
|-----------|------|-----------------|
| Summarization | Phi-3 Mini 3.8B via Ollama | Instruction-following, local, free, no cloud dependency |
| Quality Eval (LLM) | Phi-3 Mini 3.8B via Ollama | Same model, different prompt + temperature |
| Quality Eval (Rules) | Python regex + keyword matching | Fast, deterministic, complements LLM |
| Readability | textstat library | Standard, mathematical, no ML needed |
| Results Display | tabulate library | Clean terminal tables |

No cloud APIs or paid services used. All inference runs locally on CPU.
