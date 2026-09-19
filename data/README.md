# INDILEX — Dataset Documentation

## Overview

The INDILEX project processes legal judgment documents from multiple Indian High Courts. Due to their size, the full datasets are **not included in this GitHub repository**. This document describes each dataset, its structure, and how to obtain it.

## Sample Data (Included in GitHub)

### `data/sample/enriched_legal_records.xlsx`
- **Purpose**: Claude-generated ground truth annotations for model evaluation
- **Size**: ~14 KB (4 cases)
- **Columns**: `case_id`, `language`, `case_type`, `judgment_text`, `subject`, `object`, `objective_aspect`, `subjective_aspect`, `legal_provision`, `reasoning`
- **Usage**: Used as the gold standard reference in all `model_evaluation_*.ipynb` notebooks

---

## Full Datasets (Host on Hugging Face)

### Recommended HuggingFace Repository Structure

```
aniket821108/INDILEX-dataset/
│
├── README.md                           # Dataset card (copy from docs/dataset.md)
│
├── hindi/
│   ├── final_merged_hindi_cleaned.csv  # 14.6 MB — Primary cleaned Hindi dataset
│   ├── Hindi_clean_rows.csv            # 6.1 MB — Clean Hindi rows
│   └── hindi_not_clearly_specified.csv # 1.0 MB — Rows needing annotation
│
├── routed/
│   ├── INDILEX_direct_annotation.csv   # 3.1 MB — Short docs (direct route)
│   └── INDILEX_chunk_candidates.csv    # 11.5 MB — Long docs (chunk route)
│
├── regional/
│   ├── Chhattisgarh_dataset.xlsx       # 2.2 MB — Chhattisgarh HC
│   └── legal_Assamese_dataset.xlsx     # 2.6 MB — Assamese HC (Gauhati)
│
├── batch_data/                         # Batched annotation inputs
│   ├── Administrative/                 # 9 batch CSVs
│   ├── Civil/                          # batch CSVs
│   ├── Constitutional/                 # batch CSVs
│   └── Criminal/                       # 22 batch CSVs
│
├── annotations/
│   ├── Allahabad_criminal_merged.xlsx
│   ├── Allahabad_criminal_merged_gold1.xlsx
│   ├── Allahabad_criminal_merged_gold1_with_reasoning.xlsx
│   └── Allahabad_criminal_merged_gold2_with_reasoning.xlsx
│
├── extracted/
│   ├── chunk_data/                     # LLM-extracted chunk results
│   │   ├── Administrative/
│   │   ├── Civil/
│   │   ├── Constitutional/
│   │   └── Criminal/
│   └── direct_data/                    # LLM-extracted direct results
│       ├── Administrative/
│       ├── Civil/
│       ├── Constitutional/
│       └── Criminal/
│
└── raw_judgments/
    └── Chhattisgarh/                   # 201 .txt judgment files
```

---

## Dataset Descriptions

### 1. `final_merged_hindi_cleaned.csv` (Primary Dataset)
- **Purpose**: The main cleaned and merged Hindi legal judgment dataset
- **Size**: ~14.6 MB
- **Rows**: ~762 judgments
- **Columns**: `case_id`, `language`, `case_type`, `judgment_text`, `subject`, `object`, `objective_aspect`, `subjective_aspect`, `legal_provision`, `reasoning`
- **Source Courts**: Chhattisgarh, Allahabad, Delhi, MP, Jharkhand, Patna
- **Language**: Primarily Hindi
- **Used by**: All extraction and evaluation notebooks

### 2. `INDILEX_direct_annotation.csv`
- **Purpose**: Judgments routed for direct (single-pass) annotation — documents short enough for a single LLM context window
- **Size**: ~3.1 MB
- **Routing criterion**: Judgment text ≤ token threshold

### 3. `INDILEX_chunk_candidates.csv`
- **Purpose**: Long judgments routed for chunk-based multi-stage annotation
- **Size**: ~11.5 MB
- **Routing criterion**: Judgment text > token threshold
- **Used by**: `05_long_document_annotation.ipynb`

### 4. `Chhattisgarh_dataset.xlsx`
- **Purpose**: Regional court dataset from Chhattisgarh High Court
- **Size**: ~2.2 MB
- **Rows**: 201 cases

### 5. `legal_Assamese_dataset.xlsx`
- **Purpose**: Regional court dataset from Gauhati High Court (Assamese language)
- **Size**: ~2.6 MB

### 6. `batch_data/`
- **Purpose**: Pre-processed judgment data split into case-type-specific batches for production annotation
- **Case Types**: Administrative (9 batches), Civil, Constitutional, Criminal (22 batches)
- **Used by**: `04_batch_annotation.ipynb`

### 7. Allahabad Annotation Files
- **Purpose**: Gold-standard annotation iterations for Allahabad criminal cases
- Multiple versions represent annotation refinement rounds with and without reasoning

---

## How to Download

### Option A: Using `huggingface_hub` (Recommended)

```python
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="aniket821108/INDILEX-dataset",
    repo_type="dataset",
    local_dir="data/"
)
```

### Option B: Using the HuggingFace CLI

```bash
pip install huggingface_hub
huggingface-cli download aniket821108/INDILEX-dataset --repo-type dataset --local-dir data/
```

### Option C: Manual Download
Visit `https://huggingface.co/datasets/aniket821108/INDILEX-dataset` and download files individually.

---

## How the Code Uses These Datasets

| Notebook | Required Dataset(s) |
|----------|---------------------|
| `01_data_collection_pipeline` | Raw `.txt` files (Chhattisgarh/) |
| `03_dataset_audit` | `batch_data/` (all batches) |
| `04_batch_annotation` | `batch_data/` (individual batch CSVs) |
| `05_long_document_annotation` | `INDILEX_chunk_candidates.csv` |
| `06_hindi_annotation_pipeline` | `hindi_not_clearly_specified_rows.csv` |
| `07_model_selection` | `enriched_legal_records.xlsx` (sample) |
| `08_dataset_visualization` | `final_merged_hindi_cleaned.csv` |
| `experiments/extraction/*` | `enriched_legal_records.xlsx` (sample) |
| `experiments/evaluation/*` | `enriched_legal_records.xlsx` + model outputs |
