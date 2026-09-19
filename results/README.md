# INDILEX — Results

## Contents

### `figures/`
Publication-quality visualizations generated from `notebooks/08_dataset_visualization.ipynb`:

| File | Description |
|------|-------------|
| `01_case_type_distribution.png` | Distribution of Criminal/Civil/Constitutional/Administrative cases |
| `02_missing_data_heatmap.png` | Missing/empty field heatmap across annotation columns |
| `03_top_subjects.png` | Top 15 most frequent legal subjects |
| `04_top_objects.png` | Top 15 most frequent legal objects |
| `05_text_length_violin_box.png` | Judgment text character-length distribution (violin + box plot) |
| `06_word_count_histogram.png` | Word count distribution histogram |
| `07_top_objective_aspects.png` | Most common objective aspects |
| `08_casetype_remedy_heatmap.png` | Case type × remedy cross-tabulation heatmap |
| `09_legal_issues_stacked.png` | Stacked bar chart of legal issue categories |
| `10_wordclouds.png` | Word clouds per case type |
| `11_summary_dashboard.png` | Combined summary dashboard |

### `model_outputs/`
Per-model extraction results (XLSX):

| File | Model | Description |
|------|-------|-------------|
| `indilex_extracted_output_Qwen.xlsx` | Qwen2.5-7B-Instruct | Extraction on sample cases |
| `indilex_extracted_output_Qwen14B.xlsx` | Qwen2.5-14B-Instruct | Extraction on sample cases |
| `indilex_extracted_output_DeepSeek.xlsx` | DeepSeek-R1-Distill-Qwen-7B | Extraction on sample cases |
| `indilex_extracted_output_Gemma.xlsx` | Gemma-2-9B-IT | Extraction on sample cases |

### `audit_reports/`
Data quality audit CSVs from `notebooks/03_dataset_audit.ipynb` and `notebooks/09_routing_analysis.ipynb`:

| File | Description |
|------|-------------|
| `indilex_overall_audit_summary.csv` | High-level dataset statistics |
| `indilex_per_file_summary.csv` | Per-batch-file row counts and statistics |
| `indilex_per_case_type_summary.csv` | Statistics by case type |
| `indilex_row_length_audit.csv` | Per-row character/word/token length data |
| `indilex_duplicate_case_ids.csv` | Detected duplicate case IDs |
| `indilex_duplicate_judgments.csv` | Detected duplicate judgment texts |
| `indilex_annotation_missing_profile.csv` | Missing annotation field analysis |
| `indilex_load_errors.csv` | Files that failed to load |
| `case_type_routing_summary.csv` | Direct vs chunk routing distribution |
| `case_type_length_summary.csv` | Length statistics by case type |
| `annotation_missingness_by_route.csv` | Annotation completeness by route |
| `length_summary_by_route.csv` | Length distributions by route |
| `duplicate_text_rows.csv` | Duplicate text detection |
| `research_summary.csv` | Research-oriented summary statistics |
| `top_30_longest_judgments.csv` | 30 longest judgment texts |

### `evaluation_dashboard_for10_sample.png`
Combined evaluation dashboard showing multi-model comparison across all metrics.
