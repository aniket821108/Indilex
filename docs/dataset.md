# INDILEX — Dataset Card

## Dataset Description

**INDILEX** (Indian Legal Event Extraction) is a structured legal NLP dataset created by extracting and annotating information from Indian High Court judgments.

### Dataset Summary

| Property | Value |
|----------|-------|
| **Domain** | Legal NLP / Information Extraction |
| **Languages** | Hindi (primary), Assamese, English |
| **Source Courts** | Chhattisgarh HC, Allahabad HC, Delhi HC, MP HC, Jharkhand HC, Patna HC, Gauhati HC |
| **Case Types** | Criminal, Civil, Constitutional, Administrative |
| **Approximate Size** | ~762 judgments (cleaned) |
| **Annotation Method** | Hybrid (LLM + Human Verification) |
| **Annotation Schema** | 4E Framework (Entity-Event-Evidence-Evaluation) |

### Supported Tasks

- Legal Information Extraction
- Legal Text Classification (case type)
- Legal Named Entity Recognition (parties, judges, advocates)
- Legal Summarization
- Multilingual Legal NLP

### Languages

- Hindi (hi) — primary
- Assamese (as) — partial
- English (en) — partial

## Dataset Structure

### Data Fields

| Field | Type | Description |
|-------|------|-------------|
| `case_id` | string | Unique case identifier derived from filename |
| `language` | string | Detected language (hi/as/en) |
| `case_type` | categorical | Criminal / Civil / Constitutional / Administrative |
| `judgment_text` | string | Full text of the court judgment |
| `subject` | string | Legal subject / petitioner's primary claim |
| `object` | string | Object of legal action / contested matter |
| `objective_aspect` | string | Factual/objective elements of the case |
| `subjective_aspect` | string | Interpretive elements and judicial reasoning |
| `legal_provision` | string | Acts, Articles, and Sections cited |
| `reasoning` | string | Court's rationale for the decision |

### Data Splits

| Split | Description |
|-------|-------------|
| `direct` | Short judgments annotated in single pass (~3.1 MB) |
| `chunk_candidates` | Long judgments annotated via chunk pipeline (~11.5 MB) |
| `ground_truth` | Claude-annotated gold standard (4 cases) |

### Example

```json
{
  "case_id": "CRA1021_16",
  "language": "hi",
  "case_type": "Criminal",
  "judgment_text": "छत्तीसगढ़ उच्च न्यायालय, बिलासपुर ...",
  "subject": "Appellant challenging conviction under IPC Section 302",
  "object": "Murder conviction and life imprisonment sentence",
  "objective_aspect": "FIR filed, witnesses examined, forensic evidence presented",
  "subjective_aspect": "Court found prosecution's evidence credible despite minor contradictions",
  "legal_provision": "IPC Section 302, CrPC Section 374",
  "reasoning": "The court upheld conviction based on eyewitness testimony and corroborating forensic evidence..."
}
```

## Dataset Creation

### Source Data

Raw judgment text files obtained from Indian High Court websites and legal databases. Documents were processed from their original formats (`.txt` files, Excel spreadsheets) into a unified tabular schema.

### Annotation Process

1. **Automated first pass**: LLM-based annotation using Groq API with case-type-aware prompts
2. **Long-document handling**: Three-stage chunk pipeline for documents exceeding context windows
3. **Human verification**: 3 annotators verified LLM outputs across 4 case types
4. **Gap-filling**: Claude API used to fill fields initially marked "Not Clearly Specified"

### Personal and Sensitive Information

Judgment texts are **public court records** available from official court websites. They contain names of parties, judges, and advocates as part of the public legal record. No private or non-public personal information is included.

## Considerations for Using the Data

### Social Impact

This dataset supports access to justice by making legal information more searchable and analyzable. It particularly benefits Hindi-speaking populations whose legal documents are underserved by existing NLP tools.

### Limitations

- Small evaluation set (4 ground-truth cases)
- Primarily Hindi language coverage
- Limited to High Court judgments (no Supreme Court or District Courts)
- Annotation quality depends on LLM accuracy and limited human verification

### Citation

If you use this dataset, please cite:

```bibtex
@misc{indilex2026,
  title={INDILEX: Indian Legal Event Extraction Dataset and Benchmark},
  author={[Your Name]},
  year={2026},
  note={Research internship project at IIT Patna}
}
```

## Licensing

> ⚠️ License TBD — consult your institution regarding appropriate licensing.
