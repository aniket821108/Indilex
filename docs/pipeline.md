# INDILEX — Technical Pipeline Documentation

## End-to-End Pipeline

This document describes the complete technical flow of the INDILEX project, stage by stage.

---

## Stage 1: Data Collection

**Input**: Raw `.txt` court judgment files from multiple Indian High Courts

**Processing**:
- Read judgment text files from directories organized by court (Chhattisgarh, Allahabad, Delhi, Madhya Pradesh, Jharkhand, Patna)
- Extract `case_id` from filename (e.g., `CRA1021_16(23.11.23).txt` → `CRA1021_16`)
- Auto-detect language using `langdetect` library
- Create initial annotation schema with blank fields for manual/LLM annotation

**Output**: Excel files with columns: `case_id`, `language`, `case_type`, `judgment_text`, `subject`, `object`, `objective_aspect`, `subjective_aspect`, `legal_provision`, `reasoning`

**Why this stage exists**: Raw court documents are unstructured text files with no metadata. This stage creates a standardized tabular format that all downstream stages can consume.

**Key notebook**: `notebooks/01_data_collection_pipeline.ipynb`

---

## Stage 2: Judgment Metadata Extraction

**Input**: Raw `.txt` judgment files

**Processing**:
- Regex-based extraction of structured fields from judgment text headers
- Extract: case type (from filename patterns like WPC, CrlA, FA), case number, year, court name, judge names, petitioner, respondent, judgment date, citations, acts/articles mentioned, cases cited, advocate names, judgment summary, final order

**Output**: `judgments_extracted.xlsx` — structured metadata Excel file

**Why this stage exists**: Court judgments follow semi-structured patterns in their headers and conclusions. Rule-based extraction captures these patterns reliably without requiring LLM inference.

**Key notebook**: `notebooks/02_judgment_metadata_extraction.ipynb`

---

## Stage 3: Data Cleaning & Merging

**Input**: Raw data from multiple courts (Chhattisgarh XLSX, Allahabad XLSX, Delhi CSV, MP CSV, Jharkhand XLSX, Patna XLSX)

**Processing**:
- Standardize column names across courts
- Merge data from all courts into a single dataset
- Remove corrupted/empty rows
- Normalize text (whitespace, encoding)
- Identify and handle duplicates
- Language-specific cleaning for Hindi text

**Output**: `final_merged_hindi_cleaned.csv` (~762 rows, 14.6 MB)

**Why this stage exists**: Each court provides data in different formats with different column schemas. Merging creates a unified dataset for consistent annotation and evaluation.

**Key files**: `Scripts_for general_work.ipynb` (utility), `prserve/basic_script.ipynb` (inspection)

---

## Stage 4: Dataset Audit

**Input**: Merged dataset CSVs

**Processing**:
- Count rows, files, and columns
- Detect missing/empty judgments
- Identify duplicate `case_id` values
- Find exact duplicate judgment texts (using hash normalization)
- Compute character length, word count, and token count distributions
- Create length bucket histograms
- Estimate direct vs. chunk routing split
- Analyze case-type and language distributions
- Check annotation completeness per field
- Generate per-file and per-case-type summary tables

**Output**: 8 audit CSV reports in `results/audit_reports/`

**Why this stage exists**: Before designing the annotation strategy, we need to understand the data distribution — especially document lengths, which determine whether direct or chunked annotation is needed.

**Key notebook**: `notebooks/03_dataset_audit.ipynb`

---

## Stage 5: Case Routing (Direct vs. Chunk)

**Input**: Cleaned merged dataset

**Processing**:
- Compute token count for each judgment using a tokenizer
- Route short documents (≤ token threshold) to direct annotation path
- Route long documents (> token threshold) to chunk-based annotation path
- Validate that routing covers 100% of rows with no overlap

**Output**:
- `INDILEX_direct_annotation.csv` (short docs, ~3.1 MB)
- `INDILEX_chunk_candidates.csv` (long docs, ~11.5 MB)

**Why this stage exists**: LLMs have context window limits. Short documents can be annotated in a single pass, but long judgments must be split into manageable chunks for staged extraction.

**Key files**: Routing logic in `notebooks/03_dataset_audit.ipynb`, visualization in `notebooks/09_routing_analysis.ipynb`

---

## Stage 6a: Direct Annotation (Groq API)

**Input**: `batch_data/` — case-type-specific batch CSVs

**Processing**:
- Load batch CSV for a specific case type
- For each judgment row:
  - Detect case type
  - Inject case-type-specific field definitions into the Groq API prompt
  - Send judgment text to Groq (LLaMA-based) for annotation
  - Parse and validate JSON response
  - Handle errors, retries, and rate limits
- Checkpoint progress periodically (resumable on interruption)
- Generate error reports and duplicate reports

**Output**: `*_annotated.csv`, `*_errors.csv`, `*_duplicates.csv`, `*_run.log`

**Why this stage exists**: Production-scale annotation requires reliability — checkpointing, error handling, and resume capability. Using case-type-specific prompts improves annotation accuracy.

**Key notebook**: `notebooks/04_batch_annotation.ipynb`

---

## Stage 6b: Chunk-Based Annotation (vLLM + Qwen2.5-14B)

**Input**: `INDILEX_chunk_candidates.csv`

**Processing**: Three-stage pipeline:
1. **Stage A — Chunk Evidence Extraction**: Split long judgment into token-aware chunks → extract structured evidence from each chunk independently
2. **Stage B — Cross-Chunk Aggregation**: Merge evidence from all chunks → resolve conflicts → create consolidated evidence
3. **Stage C — Case-Conditioned Final Annotation**: Use aggregated evidence + case-type definitions → produce final annotation

- Uses Qwen2.5-14B-Instruct via vLLM with tensor-parallel across 2× GPUs
- Includes validation, checkpointing, dry-run mode, and batch size control

**Output**: Final annotated CSV with complete 4E schema

**Why this stage exists**: Long legal judgments (10,000+ words) exceed single-context-window limits. The three-stage architecture ensures information is captured from the entire document without losing cross-section context.

**Key notebook**: `notebooks/05_long_document_annotation.ipynb`

---

## Stage 7: Human Annotation Verification

**Input**: LLM-generated annotations

**Processing**:
- Distribute annotated cases across 3 human annotators (Aniket, Aroh, Tavishi)
- Each annotator receives a balanced set of 4 case types (Criminal, Civil, Constitutional, Administrative)
- Assignment tracked via `assignment_manifest.csv`
- Annotators verify and correct LLM outputs

**Output**: Verified annotation files per annotator per case type

**Why this stage exists**: LLM annotations require human verification to ensure quality. Multiple annotators enable inter-annotator agreement analysis.

**Key files**: `Direct_Annotation_persion/`, `chunk_Annotation_persion/` (assignment directories)

---

## Stage 8: Hindi Annotation Gap-Fill (Claude API)

**Input**: `hindi_not_clearly_specified_rows.csv` — rows where fields are marked "Not Clearly Specified"

**Processing**:
- For each row with incomplete annotations, send the full judgment text to Claude (Anthropic API)
- Claude re-reads the judgment and attempts to fill the missing fields
- Rate-limited API calls with error handling

**Output**: `hindi_annotated_output.csv`

**Why this stage exists**: Initial annotation (whether human or LLM) may leave fields as "Not Clearly Specified" when the information is ambiguous. A second-pass with a stronger model (Claude) can recover additional annotations.

**Key notebook**: `notebooks/06_hindi_annotation_pipeline.ipynb`

---

## Stage 9: LLM-Based Information Extraction (4E Framework)

**Input**: `enriched_legal_records.xlsx` (ground truth sample)

**Processing**:
- Load an instruction-tuned LLM (one of: Qwen2.5-7B, Qwen2.5-14B, DeepSeek-R1, Gemma-2-9B)
- For each judgment, construct a structured prompt requesting JSON output with 6 fields
- Run inference with JSON parsing, retries, and error handling
- Monitor performance (tokens/sec, GPU utilization)
- Save extraction results to XLSX

**Output**: `indilex_extracted_output_<model>.xlsx`

**Why this stage exists**: This is the core research experiment — benchmarking how well different open-source LLMs can extract structured legal information from unstructured judgments, without any task-specific fine-tuning.

**Key notebooks**: `notebooks/experiments/extraction/`

---

## Stage 10: Model Evaluation

**Input**: Model extraction outputs + Claude ground truth

**Processing**:
- Merge model predictions with ground truth on `case_id`
- Compute per-field metrics:
  - **Exact Match (EM)**: For categorical fields like `case_type`
  - **Token F1**: Token-level overlap for text fields
  - **ROUGE-1/2/L**: N-gram overlap for reasoning field
  - **BERTScore**: Semantic similarity for reasoning field
- Generate comparative visualizations and dashboards

**Output**: Evaluation metrics, comparison plots, dashboards

**Why this stage exists**: Quantitative evaluation against a known ground truth is essential for model selection and for reporting research results.

**Key notebooks**: `notebooks/experiments/evaluation/`, `notebooks/07_model_selection.ipynb`

---

## Stage 11: Visualization & Quality Analysis

**Input**: Final merged dataset, evaluation results

**Processing**:
- Generate 11 publication-quality figures:
  - Case type distribution
  - Missing data heatmap
  - Top subjects and objects
  - Text length distributions
  - Case-type × remedy cross-tabulation
  - Legal issues breakdown
  - Word clouds
  - Summary dashboard

**Output**: 11 PNG figures in `results/figures/`

**Why this stage exists**: Visualizations communicate dataset characteristics and results to reviewers, interviewers, and other researchers.

**Key notebook**: `notebooks/08_dataset_visualization.ipynb`
