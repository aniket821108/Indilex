# INDILEX — Indian Legal Event Extraction Dataset & Benchmark

<p align="center">
  <strong>A structured information extraction pipeline for Indian legal judgments using open-source LLMs</strong>
</p>

<p align="center">
  <a href="#overview">Overview</a> •
  <a href="#key-features">Features</a> •
  <a href="#pipeline">Pipeline</a> •
  <a href="#models">Models</a> •
  <a href="#results">Results</a> •
  <a href="#installation">Installation</a> •
  <a href="#usage">Usage</a>
</p>

---

## Overview

**INDILEX** (Indian Legal Event Extraction) is an end-to-end NLP pipeline for extracting structured legal information from Indian High Court judgments. The project processes raw court judgment text files from multiple Indian states (Chhattisgarh, Allahabad, Delhi, Madhya Pradesh, Jharkhand, Patna) and multiple languages (Hindi, Assamese, English), transforming unstructured legal documents into a structured annotation dataset.

The pipeline leverages multiple open-source instruction-tuned LLMs to automatically extract legal event tuples following a **4E (Entity-Event-Evidence-Evaluation) Framework**, then rigorously evaluates extraction quality against Claude-generated ground truth using NLP metrics.

## Problem Statement

Indian legal documents are voluminous, multilingual, and lack standardized structure. Manual extraction of legal information — parties involved, legal provisions cited, reasoning, and case outcomes — is extremely time-consuming and not scalable. INDILEX addresses this by building an automated pipeline that:

1. **Standardizes** raw judgment text across courts and languages
2. **Routes** documents by length (direct annotation vs. chunk-based annotation)
3. **Extracts** structured information using LLMs with case-type-aware prompting
4. **Evaluates** extraction quality with multiple NLP metrics
5. **Produces** a benchmark dataset for legal NLP research

## Key Features

- **Multi-court, multilingual data processing** — Hindi, Assamese, and English judgments from 6+ Indian High Courts
- **Intelligent document routing** — Automatic classification of documents into direct (short) and chunk-based (long) annotation paths based on token length
- **Case-type-aware annotation** — Specialized prompt templates for Criminal, Civil, Constitutional, and Administrative cases
- **4E Framework extraction** — Structured extraction of `case_type`, `subject`, `object`, `objective_aspect`, `subjective_aspect`, and `reasoning`
- **Multi-model benchmarking** — Comparative evaluation across 4 open-source LLMs
- **Production-grade batch processing** — Checkpointed, resumable annotation with error handling
- **Comprehensive data quality auditing** — Length analysis, duplicate detection, annotation completeness checks
- **Three-stage long-document pipeline** — Chunk → Evidence Extraction → Aggregation → Final Annotation for documents exceeding context windows

## Pipeline

```
Raw Legal Judgment Texts (.txt — Chhattisgarh, Allahabad, Delhi, MP, Patna, Jharkhand)
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 1: Data Collection        │
            │   Multi-court merging & cleanup   │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 2: Data Cleaning          │
            │   Language detection, dedup,      │
            │   normalization                   │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 3: Dataset Audit          │
            │   Length distributions, routing   │
            │   estimates, quality checks       │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 4: Case Routing           │
            │   Short → Direct   Long → Chunk   │
            └──────┬────────────────┬───────────┘
                   ↓                ↓
     ┌──────────────────┐  ┌────────────────────────┐
     │ Stage 5a: Direct │  │ Stage 5b: Chunk        │
     │ Annotation       │  │ Annotation (3-stage)   │
     │ (Groq API)       │  │ (vLLM / Qwen2.5-14B)  │
     └────────┬─────────┘  └──────────┬─────────────┘
              ↓                       ↓
            ┌───────────────────────────────────┐
            │   Stage 6: Human Verification     │
            │   3 annotators × 4 case types     │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 7: LLM Information        │
            │   Extraction (4E Framework)       │
            │   Qwen | Gemma | DeepSeek         │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 8: Model Evaluation       │
            │   EM, F1, ROUGE, BERTScore        │
            │   vs. Claude Ground Truth         │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 9: Model Selection        │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 10: Visualization &       │
            │   Quality Analysis                │
            └───────────────┬───────────────────┘
                            ↓
            ┌───────────────────────────────────┐
            │   Stage 11: Final Annotated       │
            │   Dataset                         │
            └───────────────────────────────────┘
```

## Dataset

### Source Data
- **Courts**: Chhattisgarh HC, Allahabad HC, Delhi HC, Madhya Pradesh HC, Jharkhand HC, Patna HC, Gauhati HC
- **Languages**: Hindi (primary), Assamese, English
- **Case Types**: Criminal, Civil, Constitutional, Administrative
- **Format**: Raw `.txt` judgment files processed into structured CSVs

### Annotation Schema (4E Framework)

| Field | Description |
|-------|-------------|
| `case_id` | Unique identifier derived from filename |
| `language` | Auto-detected language of the judgment |
| `case_type` | Criminal / Civil / Constitutional / Administrative |
| `judgment_text` | Full text of the court judgment |
| `subject` | Primary legal subject / petitioner's claim |
| `object` | Object of the legal action / what is being contested |
| `objective_aspect` | Factual/objective elements of the case |
| `subjective_aspect` | Interpretive/subjective elements and judicial reasoning |
| `legal_provision` | Acts, Articles, and Sections cited |
| `reasoning` | Court's rationale for the decision |

### Dataset Hosting

Large datasets are hosted separately on Hugging Face. See [`data/README.md`](data/README.md) for download instructions.

## Models / Approaches

Four open-source instruction-tuned LLMs were benchmarked for legal information extraction:

| Model | Parameters | Source | GPU Memory |
|-------|-----------|--------|------------|
| **Qwen2.5-7B-Instruct** | 7B | Alibaba | ~14 GB (fp16) |
| **Qwen2.5-14B-Instruct** | 14B | Alibaba | ~28 GB (fp16, 2×GPU) |
| **Gemma-2-9B-IT** | 9B | Google | ~18 GB (fp16) |
| **DeepSeek-R1-Distill-Qwen-7B** | 7B | DeepSeek | ~14 GB (fp16) |

**Ground Truth**: Claude (Anthropic) annotations used as reference for evaluation.

**Annotation API**: Groq API used for production batch annotation with case-type-aware prompts.

## Experiments and Evaluation

Each model was evaluated against Claude ground truth using:

- **Exact Match (EM) Accuracy** — for categorical fields (`case_type`)
- **Token F1 Score** — token-level overlap for text fields
- **ROUGE-1 / ROUGE-2 / ROUGE-L** — n-gram overlap for reasoning fields
- **BERTScore** — semantic similarity for reasoning fields

Evaluation notebooks with full results are in [`notebooks/experiments/evaluation/`](notebooks/experiments/evaluation/).

## Results

### Visualizations

The project produces 11 publication-quality visualizations covering:

| Figure | Content |
|--------|---------|
| Case type distribution | Breakdown of Criminal/Civil/Constitutional/Administrative |
| Missing data heatmap | Annotation completeness across fields |
| Top subjects & objects | Most frequent legal subjects and objects |
| Text length analysis | Violin/box plots of judgment lengths |
| Word count distribution | Histogram of word counts |
| Case-type × remedy heatmap | Cross-tabulation analysis |
| Legal issues stacked chart | Distribution of legal issues |
| Word clouds | Visual text analysis per case type |
| Summary dashboard | Combined overview dashboard |

Selected figures are available in [`results/figures/`](results/figures/).

## Repository Structure

```
INDILEX/
├── README.md                              # This file
├── LICENSE                                # License
├── requirements.txt                       # Python dependencies
├── .gitignore                             # Git exclusions
│
├── notebooks/                             # Organized Jupyter notebooks
│   ├── 01_data_collection_pipeline.ipynb  # Raw .txt → initial annotation Excel
│   ├── 02_judgment_metadata_extraction.ipynb  # Structured metadata extraction
│   ├── 03_dataset_audit.ipynb             # Length audit, routing estimates
│   ├── 04_batch_annotation.ipynb          # Groq API batch annotation
│   ├── 05_long_document_annotation.ipynb  # 3-stage chunk annotation (vLLM)
│   ├── 06_hindi_annotation_pipeline.ipynb # Claude API Hindi annotation fill
│   ├── 07_model_selection.ipynb           # Model comparison & selection
│   ├── 08_dataset_visualization.ipynb     # 11 publication-quality figures
│   ├── 09_routing_analysis.ipynb          # Direct vs Chunk visualization
│   └── experiments/
│       ├── extraction/                    # Per-model extraction notebooks
│       │   ├── qwen2.5_7b_extraction.ipynb
│       │   ├── qwen2.5_14b_extraction.ipynb
│       │   ├── deepseek_r1_extraction.ipynb
│       │   └── gemma2_9b_extraction.ipynb
│       └── evaluation/                   # Per-model evaluation notebooks
│           ├── qwen2.5_7b_evaluation.ipynb
│           ├── qwen2.5_14b_evaluation.ipynb
│           ├── deepseek_r1_evaluation.ipynb
│           └── gemma2_9b_evaluation.ipynb
│
├── data/
│   ├── sample/                            # Small sample data for demos
│   │   └── enriched_legal_records.xlsx    # Claude ground truth (4 cases)
│   └── README.md                          # Dataset documentation & download guide
│
├── results/
│   ├── figures/                           # 11 publication-quality PNG figures
│   ├── model_outputs/                     # Per-model extraction XLSX files
│   ├── audit_reports/                     # Data quality audit CSVs
│   ├── evaluation_dashboard_for10_sample.png
│   └── README.md                          # Results documentation
│
└── docs/
    ├── pipeline.md                        # Full technical pipeline documentation
    ├── dataset.md                         # Dataset card
    └── interview_notes.md                 # Interview preparation guide
```

## Installation

```bash
# Clone the repository
git clone https://github.com/<your-username>/INDILEX.git
cd INDILEX

# Create virtual environment (recommended)
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or: venv\Scripts\activate  # Windows

# Install dependencies
pip install -r requirements.txt
```

### Hardware Requirements

- **For extraction experiments**: NVIDIA GPU with ≥24 GB VRAM (e.g., RTX 3090)
- **For Qwen2.5-14B**: 2× NVIDIA GPUs with ≥24 GB each (tensor parallel)
- **For evaluation/visualization**: CPU-only is sufficient

## Usage

### 1. Prepare Data

Download datasets from HuggingFace (see [`data/README.md`](data/README.md)):

```bash
# Install huggingface_hub
pip install huggingface_hub

# Download dataset
# huggingface-cli download <your-username>/INDILEX-dataset --local-dir data/
```

### 2. Run the Pipeline

```bash
# Step 1: Data Collection — process raw .txt files
jupyter notebook notebooks/01_data_collection_pipeline.ipynb

# Step 2: Dataset Audit — analyze length distributions
jupyter notebook notebooks/03_dataset_audit.ipynb

# Step 3: Batch Annotation — annotate using Groq API
jupyter notebook notebooks/04_batch_annotation.ipynb

# Step 4: Run LLM Extraction (e.g., Qwen2.5-7B)
jupyter notebook notebooks/experiments/extraction/qwen2.5_7b_extraction.ipynb

# Step 5: Evaluate Results
jupyter notebook notebooks/experiments/evaluation/qwen2.5_7b_evaluation.ipynb

# Step 6: Generate Visualizations
jupyter notebook notebooks/08_dataset_visualization.ipynb
```

### 3. Quick Demo

To quickly explore the project:

```python
import pandas as pd

# Load Claude ground truth sample
gt = pd.read_excel("data/sample/enriched_legal_records.xlsx")
print(gt.columns.tolist())
print(gt[["case_id", "case_type", "subject"]].head())

# Load a model's extraction output
qwen = pd.read_excel("results/model_outputs/indilex_extracted_output_Qwen.xlsx")
print(qwen[["case_id", "case_type", "subject"]].head())
```

## Limitations

- **Small evaluation set**: Model evaluation uses a limited number of ground-truth cases annotated by Claude, which may not capture the full diversity of Indian legal documents.
- **Language coverage**: Primary focus is Hindi judgments; Assamese and English support is partial.
- **No fine-tuning**: All models are used in zero-shot/few-shot mode with prompt engineering only — no task-specific fine-tuning was performed.
- **Context window constraints**: Long judgments exceeding model context windows require chunking, which may lose cross-section context.
- **Ground truth dependency**: Evaluation relies on Claude as ground truth, which itself may contain extraction errors.

## Future Improvements

- **Expand language coverage** to include more Indian regional languages (Bengali, Tamil, Telugu, etc.)
- **Fine-tune models** on the annotated INDILEX dataset for improved domain-specific extraction
- **Implement cross-document analysis** to identify citation networks and precedent patterns
- **Build a retrieval-augmented generation (RAG) pipeline** for legal question answering
- **Scale to Supreme Court** and District Court judgments
- **Add inter-annotator agreement metrics** for human verification quality
- **Integrate with legal databases** (IndianKanoon, SCI) for automated data collection

## Dataset Hosting

Large datasets (>50 MB total) are not included in this GitHub repository. They should be hosted on Hugging Face:

```
HuggingFace: <your-username>/INDILEX-dataset
├── final_merged_hindi_cleaned.csv       # Primary Hindi dataset
├── INDILEX_chunk_candidates.csv         # Chunk-route candidates
├── INDILEX_direct_annotation.csv        # Direct-route annotations
├── Chhattisgarh_dataset.xlsx            # Chhattisgarh court data
├── legal_Assamese_dataset.xlsx          # Assamese court data
├── batch_data/                          # Batched annotation data
├── extracted_chunk_data/                # Chunk extraction results
└── extracted_direct_data/               # Direct extraction results
```

See [`data/README.md`](data/README.md) for full details.

## License

> ⚠️ **License TBD** — Please consult your institution (IIT Patna) regarding appropriate licensing for this research project before choosing a license.

## Author

**[Your Name]**
- 🎓 B.Tech, 4th Year — [Your University]
- 🔬 Research Intern — IIT Patna
- 🐙 GitHub: [your-github-username](https://github.com/your-github-username)
- 💼 LinkedIn: [your-linkedin](https://linkedin.com/in/your-linkedin)

---

<p align="center">
  <em>Built during a research internship at IIT Patna, 2026</em>
</p>
