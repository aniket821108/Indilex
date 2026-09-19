# INDILEX — Interview Preparation Guide

> **Use this document to prepare for placement interviews.** Every answer is derived from the actual INDILEX project — not generic ML boilerplate.

---

## 1. One-Line Explanation

"INDILEX is an end-to-end NLP pipeline that extracts structured legal information from Indian High Court judgments using open-source LLMs, and benchmarks four models against Claude ground truth."

## 2. 30-Second Explanation

"INDILEX tackles the problem of extracting structured information from unstructured Indian court judgments. I built a pipeline that takes raw Hindi legal documents from 6+ High Courts, cleans and merges them, routes them based on document length — short docs go through direct LLM annotation, long docs go through a three-stage chunk-based pipeline. I then benchmarked four open-source models — Qwen, Gemma, and DeepSeek — against Claude-generated ground truth, using metrics like ROUGE, BERTScore, and Token F1. The whole system includes production-grade batch processing with checkpointing and error recovery."

## 3. 1-Minute Explanation

"Indian legal judgments are long, multilingual, unstructured documents. Manually extracting who's suing whom, under which legal provisions, and what the court's reasoning was — that doesn't scale. INDILEX automates this.

I collected raw judgment text files from courts across India — Chhattisgarh, Allahabad, Delhi, and others — primarily in Hindi. I built a data pipeline that merges data from different courts, each with different formats, into a unified schema. Then I did a thorough audit — analyzing document lengths, duplicates, and annotation gaps.

The key engineering challenge was that some judgments are 50,000+ characters. An LLM can't process that in one shot. So I built a routing system: short documents get annotated directly via the Groq API, and long documents go through a three-stage chunk pipeline — first extracting evidence from each chunk, then aggregating across chunks, then producing a final structured annotation.

For the research component, I ran the same extraction task using four different open-source LLMs — Qwen2.5-7B, Qwen2.5-14B, DeepSeek-R1, and Gemma-2-9B — and evaluated them against Claude annotations using Exact Match, Token F1, ROUGE, and BERTScore. The evaluation helps identify which model is best for legal text extraction without fine-tuning."

## 4. 2-Minute Technical Explanation

"Let me walk through the technical architecture.

**Data layer**: Raw judgment `.txt` files from 6 Indian High Courts. Each court has its own format — Chhattisgarh provides structured Excel with case metadata, Allahabad provides raw text files. I built a merger that standardizes schemas, detects languages using langdetect, handles encoding issues, and produces a unified CSV with ~762 judgments.

**Audit layer**: Before annotation, I audit the dataset — computing character/word/token distributions, finding duplicates via hash normalization, and generating routing estimates. This drives the direct vs. chunk split.

**Annotation layer — Direct path**: For short documents, I use the Groq API with LLaMA models. The system is production-grade — it detects each row's case type, injects case-type-specific definitions into the prompt (Criminal cases have different annotation fields than Civil ones), handles rate limits, checkpoints progress to CSV, logs every request, and can resume from interruption.

**Annotation layer — Chunk path**: For long documents, I built a three-stage pipeline using Qwen2.5-14B via vLLM across 2 GPUs in tensor-parallel mode. Stage A: token-aware chunking + per-chunk evidence extraction. Stage B: cross-chunk evidence aggregation — merging information that might be split across chunk boundaries. Stage C: final annotation using aggregated evidence plus case-type definitions.

**Extraction experiments**: Using a 4E (Entity-Event-Evidence-Evaluation) framework, I ran the same extraction on 4 models — Qwen2.5-7B-Instruct, Qwen2.5-14B-Instruct, DeepSeek-R1-Distill-Qwen-7B, and Gemma-2-9B-IT. Each model reads the judgment text and produces a JSON with 6 fields: case_type, subject, object, objective_aspect, subjective_aspect, and reasoning.

**Evaluation**: I compare each model's output against Claude ground truth using: Exact Match for categorical fields, Token F1 for text overlap, ROUGE-1/2/L for reasoning quality, and BERTScore for semantic similarity. Results show the relative strengths of each model on different legal text extraction subtasks.

**Human verification**: I distributed annotation tasks to 3 human annotators across 4 case types, tracked via assignment manifests. This creates a human-verified reference layer."

---

## 5. Problem Statement

**The problem**: India's High Courts produce thousands of judgments annually in multiple languages. These judgments contain critical legal information — parties, provisions, reasoning — buried in unstructured text. No scalable system exists to automatically extract and structure this information.

**Why it matters**: Structured legal data enables:
- Legal research and precedent analysis
- Access to justice (making legal information searchable)
- Automated legal analytics for law firms and courts
- NLP research on under-resourced languages (Hindi, Assamese)

## 6. Why This Problem Matters

- India has **25 High Courts** generating millions of judgments — virtually none are structured
- Most Indian legal documents are in **regional languages** — NLP tools for Hindi/Assamese legal text barely exist
- Manual annotation costs **$50-100 per document** at scale — automated extraction is orders of magnitude cheaper
- The **4E framework** provides a reusable schema for legal event extraction across jurisdictions

## 7. Dataset Explanation

"Our dataset comes from 6 Indian High Courts — Chhattisgarh, Allahabad, Delhi, Madhya Pradesh, Jharkhand, and Patna. The primary language is Hindi, with some Assamese and English judgments. We have approximately 762 judgments after cleaning and deduplication.

Each judgment is annotated with 6 structured fields following the 4E framework:
- **case_type**: Criminal, Civil, Constitutional, or Administrative
- **subject**: Who or what is bringing the case
- **object**: What is being contested
- **objective_aspect**: Factual/objective elements
- **subjective_aspect**: Interpretive elements and judicial reasoning
- **reasoning**: The court's rationale for its decision

The annotation includes both LLM-generated and human-verified labels."

## 8. Data Preprocessing

"Preprocessing was non-trivial because data came from 6 courts in 3 formats:
- **Chhattisgarh**: Structured XLSX with 10 columns already filled
- **Allahabad/Delhi/MP**: Raw `.txt` files requiring regex-based metadata extraction
- **Jharkhand/Patna**: Semi-structured Excel

Steps:
1. Standardize column names across all sources
2. Language detection on every judgment
3. Text normalization: Unicode normalization, whitespace collapsing, encoding fixes
4. Duplicate detection using normalized text hashing
5. Remove corrupted/empty rows
6. Merge into a unified 762-row dataset"

## 9. Why Chunking Was Used

"Indian legal judgments can be extremely long — some exceed 50,000 characters or 15,000 tokens. Most LLMs have context windows of 4K-8K tokens (or 32K for larger models, but with degraded performance on long sequences).

Rather than truncating judgments (which loses critical information from later sections like the final order), I implemented token-aware chunking:
- The audit stage computes per-document token counts
- Documents below the threshold go to direct annotation (single LLM call)
- Documents above go through a three-stage chunk pipeline

The chunk pipeline is specifically designed to prevent information loss at chunk boundaries — Stage B aggregates evidence across chunks, resolving cases where a party name appears in chunk 1 but the relevant legal provision appears in chunk 3."

## 10. Why Annotation Was Necessary

"The goal of INDILEX is to create a structured legal dataset. Raw judgments are just text — annotation transforms them into structured records that can be queried, analyzed, and used for training future models.

We use a hybrid annotation approach:
1. **LLM-first**: Automated annotation using Groq API (faster, cheaper)
2. **Human verification**: 3 annotators verify LLM outputs (ensures quality)
3. **Gap-filling**: Claude API re-annotates fields marked 'Not Clearly Specified'

This is more efficient than pure manual annotation while maintaining quality."

## 11. How Information Extraction Works

"Each LLM receives a structured prompt containing:
1. The full judgment text
2. A task description explaining the 4E framework
3. Definitions of each output field
4. Instructions to produce strict JSON output

The prompt is case-type-aware — Criminal cases get Criminal-specific field definitions (e.g., 'charges', 'sentence'), while Civil cases get Civil-specific ones.

The model generates a JSON response with 6 fields. I implemented robust parsing with:
- Regex-based JSON extraction from model output
- Retry logic (up to 3 attempts per judgment)
- Fallback to partial extraction if JSON is malformed
- Error logging for failed cases"

## 12. Why LLMs Were Used (Not Traditional NLP)

"Traditional NLP approaches (NER, rule-based extraction, keyword matching) work for simple fields like dates and case numbers. But legal reasoning, subjective aspects, and complex legal relationships require:
- **Understanding context** across multiple paragraphs
- **Inferring implicit information** (e.g., when a case type isn't explicitly stated)
- **Generating natural language summaries** of complex reasoning

Instruction-tuned LLMs are the only current approach that handles all three. I chose open-source models specifically for reproducibility — anyone with a GPU can run this pipeline."

## 13. Models Tested

| Model | Why Chosen | Strengths |
|-------|-----------|-----------|
| **Qwen2.5-7B-Instruct** | Strong multilingual capabilities, Alibaba's top open model | Good Hindi understanding, efficient |
| **Qwen2.5-14B-Instruct** | Larger Qwen variant for quality comparison | Better reasoning at cost of GPU memory |
| **DeepSeek-R1-Distill-Qwen-7B** | Reasoning-specialized distilled model | Enhanced chain-of-thought reasoning |
| **Gemma-2-9B-IT** | Google's instruction-tuned model | Different architecture perspective |

All models run via HuggingFace Transformers in fp16 on NVIDIA RTX 3090 GPUs.

## 14. Evaluation Methodology

"I use Claude-generated annotations as ground truth — Claude is the strongest available model and serves as our reference.

**Metrics**:
- **Exact Match (EM)**: Does `case_type` match exactly? Binary.
- **Token F1**: Tokenize both prediction and ground truth, compute precision/recall/F1 on token sets. Captures partial matches.
- **ROUGE-1/2/L**: N-gram overlap for reasoning — measures how much of the ground truth reasoning the model captured.
- **BERTScore**: Uses contextual embeddings to measure semantic similarity — catches paraphrases that ROUGE misses.

**Why multiple metrics?**: EM is too strict for text fields. Token F1 captures partial matches but misses synonyms. ROUGE captures n-gram patterns. BERTScore captures meaning. Together, they give a comprehensive picture."

## 15. Data Quality Checks

"I implemented comprehensive auditing:
- **Duplicate detection**: Both case_id duplicates and content-hash duplicates
- **Length distribution analysis**: Identifies unusually short/long judgments (short = potentially corrupted, long = need chunking)
- **Annotation completeness**: Percentage of non-empty fields per case type
- **Per-batch quality**: Error rates, parse failures, retry counts in the annotation pipeline
- **Routing validation**: Ensure direct + chunk = 100% of dataset with no overlap

All audit results are saved as CSVs in `results/audit_reports/` for reproducibility."

## 16. Challenges Faced

1. **Context window limitations**: Solved with three-stage chunk pipeline
2. **JSON parsing failures**: LLMs sometimes generate malformed JSON — implemented regex-based extraction with fallbacks
3. **Multi-court schema heterogeneity**: Each court uses different column names/formats — built a unified merger
4. **Hindi text processing**: Standard NLP tools (tokenizers, length estimators) behave differently with Devanagari script — adapted token counting
5. **GPU memory management**: Qwen2.5-14B needs 2× RTX 3090 — implemented tensor-parallel deployment via vLLM
6. **Rate limiting**: Groq API has request limits — implemented exponential backoff and checkpointing
7. **Annotation gap-filling**: Initial annotations often left fields as "Not Clearly Specified" — built a second-pass Claude pipeline

## 17. Engineering Solutions & Decisions

| Challenge | Decision | Rationale |
|-----------|----------|-----------|
| Long documents | Three-stage chunk pipeline | Preserves cross-section context |
| Production reliability | Checkpoint + resume system | Annotation jobs can run for hours |
| Case-type diversity | Case-type-aware prompts | Criminal and Civil cases need different field definitions |
| Model comparison | Same prompt template across models | Fair apples-to-apples comparison |
| Ground truth | Claude as reference | Strongest available model for annotation |
| Batch processing | Per-batch CSVs with error isolation | One bad row doesn't crash the pipeline |
| Multi-GPU | vLLM tensor-parallel | Only viable approach for 14B model on 24GB GPUs |

## 18. What I Personally Worked On

> ⚠️ **Customize this section with your actual contributions before the interview.**

- Built the end-to-end data pipeline from raw `.txt` files to structured datasets
- Implemented the case-routing system (direct vs. chunk annotation paths)
- Developed the production batch annotation system with checkpointing
- Ran extraction experiments across 4 LLMs on GPU cluster
- Built the evaluation framework (EM, F1, ROUGE, BERTScore)
- Created data quality audit pipeline
- Generated publication-quality visualizations
- Managed human annotation distribution across 3 annotators
- Designed the three-stage long-document annotation architecture

## 19. What Can Be Improved

- **Fine-tune models**: Currently using zero-shot — fine-tuning on the annotated data would significantly improve extraction quality
- **Inter-annotator agreement**: Need formal IAA metrics (Cohen's kappa, Fleiss' kappa) across the 3 human annotators
- **Scale to more courts**: Currently 6 courts — India has 25 High Courts + Supreme Court + District Courts
- **Add more languages**: Bengali, Tamil, Telugu, Marathi cover most remaining Indian legal text
- **RAG pipeline**: Retrieval-Augmented Generation for legal question answering over the structured dataset
- **Automated benchmarking**: CI/CD pipeline that re-evaluates when new models are released
- **Cross-document analysis**: Citation networks, precedent tracking, temporal analysis of legal reasoning

---

## 20. Likely Interviewer Questions & Strong Answers

### Q: "What is INDILEX in one sentence?"
**A**: "It's a multilingual legal NLP pipeline that automatically extracts structured information from Indian court judgments using open-source LLMs and evaluates extraction quality against Claude ground truth."

### Q: "Why not just use ChatGPT/Claude directly?"
**A**: "Three reasons: (1) Cost — annotating 762+ judgments via commercial APIs is expensive; open-source models run on our own GPUs. (2) Reproducibility — open-source models can be deployed anywhere without API dependencies. (3) Research value — benchmarking multiple architectures reveals which model characteristics matter for legal text."

### Q: "How do you handle long documents that exceed context windows?"
**A**: "I built a three-stage pipeline: first, split the document into token-aware chunks. Then extract evidence from each chunk independently. Then aggregate evidence across chunks — resolving cases where relevant information spans chunk boundaries. Finally, produce a single structured annotation using the aggregated evidence."

### Q: "What metrics did you use and why?"
**A**: "Four complementary metrics: Exact Match for categorical fields, Token F1 for partial text overlap, ROUGE for n-gram coverage of reasoning, and BERTScore for semantic similarity. Each metric captures a different aspect — using only one would miss important quality signals."

### Q: "What was the hardest technical challenge?"
**A**: "JSON parsing from LLM outputs. These models aren't trained to produce strict JSON — they often add markdown formatting, extra text, or escape characters. I built a robust parser that uses regex to extract JSON blocks, handles partial outputs, retries failed extractions with modified prompts, and logs failures for debugging. About 5-8% of extractions needed retries."

### Q: "How did you ensure data quality?"
**A**: "Multi-layered: (1) Automated audit checking duplicates, lengths, and completeness. (2) LLM-generated initial annotations. (3) Human verification by 3 annotators. (4) Claude-based gap-filling for ambiguous fields. (5) Quantitative evaluation against ground truth."

### Q: "What's the 4E Framework?"
**A**: "4E stands for Entity, Event, Evidence, and Evaluation. It's the annotation schema — Entity captures the parties (subject, object), Event captures what happened (case_type, objective_aspect), Evidence captures the legal basis (legal_provision, reasoning), and Evaluation is the assessment of extraction quality."

### Q: "Which model performed best?"
**A**: "Refer to the actual evaluation results in `notebooks/experiments/evaluation/` — I'd recommend running the notebooks and noting the specific metrics before your interview, so you can cite exact numbers."

### Q: "Why Hindi? Why not English?"
**A**: "Most Indian NLP research focuses on English legal text. But the vast majority of High Court judgments in states like Chhattisgarh, MP, and UP are in Hindi. There's a critical resource gap — INDILEX specifically targets this underserved domain."

### Q: "How is this different from existing legal NLP datasets?"
**A**: "Existing datasets like CaseLaw Access Project or ILSI focus on English. INDILEX is (1) multilingual — Hindi, Assamese, English, (2) covers 6+ Indian High Courts, (3) uses a structured 4E annotation schema, and (4) includes benchmarking of 4 open-source LLMs. It's specifically designed for Indian legal NLP research."

### Q: "What would you do with 6 more months?"
**A**: "Three things: (1) Fine-tune Qwen2.5-7B on the annotated data — I expect 15-25% improvement on reasoning quality. (2) Build a RAG pipeline for legal question answering. (3) Scale to 5,000+ judgments across all 25 High Courts."

### Q: "Explain the system architecture."
**A**: "Modular pipeline: Data Collection → Cleaning → Audit → Routing → Annotation (two paths) → Human Verification → LLM Extraction → Evaluation → Visualization. Each stage is an independent notebook that reads from and writes to well-defined data files. No stage depends on another's runtime state — you can re-run any stage independently."

### Q: "What technologies did you use?"
**A**: "Python ecosystem: pandas for data processing, HuggingFace Transformers for model loading, vLLM for high-throughput inference, Groq and Anthropic APIs for annotation, ROUGE-score and BERTScore for evaluation, matplotlib/seaborn for visualization. Infrastructure: 2× NVIDIA RTX 3090 GPUs, Jupyter notebooks, Git for version control."
