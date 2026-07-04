# US Treasury data project

Built a RAG system to answer questions about US Treasury records. Did a baseline version (basic/naive) then an engineered version (better chunking + metadata filtering) to see how much it actually improves things.

## Setup

- **Data**: Databricks OfficeQA dataset (Treasury Bulletins, from Hugging Face)
- **Years used**: 2018–2025 (went a bit wider than the minimum 4 to get more workable questions — more on that below)
- **Question set**: 11 questions from `officeqa_full.csv`, filtered down to only ones where ALL required source docs fall inside my year range
- **Vector DB**: ChromaDB
- **Embedding model**: sentence-transformers (`all-MiniLM-L6-v2`)
- **LLM**: DeepSeek (`deepseek-chat`)

## Architecture

**Baseline:**
- Naive fixed-size chunks (1000 chars, no overlap)
- No metadata, just raw text chunks
- 8,873 chunks total

**Engineered:**
- Bigger chunks with overlap (2000 chars, 300 overlap) so tables don't get sliced apart
- Tagged every chunk with Year + Month metadata pulled from the filename
- Retrieval filters by year (extracted from the question text) before searching, falls back to unfiltered if no year found
- 5,226 chunks total

## Scorecard

| Metric | Baseline | Engineered |
|---|---|---|
| Hit Rate@5 | 18.18% | **63.64%** |
| MRR | 0.182 | **0.439** |
| Groundedness | N/A (no claims made) | 50% |
| Factual Accuracy | 0.00% | **9.09%** |
| Hallucination Rate | N/A | 50% |

## Reflection

**1. Where was the bottleneck in baseline — retrieval or generation?**

Retrieval, no question. Hit Rate@5 was only 18%, meaning the right document barely ever showed up in the top 5 results. Because of that, Factual Accuracy was 0% — but that's not really a generation failure, since the model correctly said "I don't know" instead of hallucinating when it didn't have the right info. It never even got a fair shot to answer.

**2. Did the metadata fix help retrieval more than generation?**

Yeah, way more. Adding year metadata + filtering by it before searching took Hit Rate@5 from 18% → 64% and MRR from 0.18 → 0.44. That's almost entirely a retrieval win. Generation improved too (0% → 9% accuracy, plus we finally got groundable claims at all — 50% groundedness where baseline had none) but the bigger lift was clearly on the retrieval side.

**3. What breaks first if scaled to the full 80-year archive (1939–2025)?**

Honestly, probably my year-extraction trick first. Right now I just regex out any 4-digit year mentioned in the question. That falls apart once questions reference multiple years, relative time ("5 years prior"), or no year at all — and I already saw this in my own filtering: 5 questions in the full dataset needed docs spanning multiple years, which a single-year filter can't handle. Past that, going from 31 files to 697 (many of which are pre-1996 scans needing OCR) would blow up chunk volume and probably force a move off in-memory ChromaDB to something more production-grade.

## Notes / Honesty Check

- Only 11 questions in my final test set (after filtering to my year range) — small sample, so these percentages are directional, not statistically bulletproof.
- Data came from Hugging Face (`databricks/officeqa`) — the GitHub repo only has code, the CSVs/text/PDFs live on HF now (gated, but access is basically instant).
