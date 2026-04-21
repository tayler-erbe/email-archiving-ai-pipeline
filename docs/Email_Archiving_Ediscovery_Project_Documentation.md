# Email Archiving and E-Discovery Using Word Frequency Analysis
*Project Documentation and Technical Reference*

---

## 1. Project Overview

This project focuses on email archiving and e-discovery with the goal of making historical digital correspondence more accessible to researchers while protecting sensitive data. The core approach employs word frequency analysis to redact infrequently appearing words, which has shown promising results in balancing readability with privacy.

The work originated from a collaboration between the University Library, Records and Information Management Services (RIMS), and the Decision Support team, with the aim of developing a scalable, automated pipeline for processing unstructured digital archives left behind by former university faculty and staff.

---

## 2. Project Background and Motivation

RIMS had been working with email archiving and e-discovery tools for several years to help archives make historical digital correspondence available to researchers more quickly and with less effort, while simultaneously protecting sensitive data.

The team developed a strategy employing word frequency to redact infrequently appearing words. Initial testing showed promising results in balancing readability with privacy. The collaboration was initiated to explore whether advanced data science techniques could improve or streamline this method.

The broader institutional motivation stems from a recurring challenge: digital files left behind by retired, deceased, or former university faculty and staff arrive in bulk, in mixed formats, often decades old, lacking context or metadata, and potentially containing sensitive personal information. Manual processing is not scalable, and content often sits untouched for years.

---

## 3. Project Kick-Off Meeting Summary

The following summarizes the initial planning meeting held to scope the Digital Archives Processing Initiative.

### 3.1 Discussion Overview

The team discussed a potential project focused on organizing and processing unstructured digital files left behind by retired, deceased, or former university faculty and staff. The goal is to develop an automated pipeline that can make these archives accessible, searchable, and research-ready.

### 3.2 Key Challenges Identified

- Files arrive in bulk and include mixed formats (Word, PDFs, images, videos, etc.).
- Materials may be decades old and lack context or metadata.
- Sensitive personal information (e.g., medical or financial records) may be hidden within the data.
- Manual processing is not scalable; content often sits untouched for years.

### 3.3 Proposed Automated Pipeline

The team outlined a multi-step automated pipeline with the following stages:

- **Ingest:** Accept all file types from nested folders.
- **Extract:** Parse readable content (text, OCR for images, etc.).
- **Convert:** Produce a structured data table including file paths and content previews.
- **Summarize:** Generate per-file content summaries and redact sensitive information.
- **Embed and Vectorize:** Enable semantic search functionality across all content.
- **Query:** Allow users to find files or summaries based on topics, people, or research themes.
- **Deduplicate:** Identify duplicates, outdated versions, or non-relevant materials.
- **Finding Aids:** Generate human-readable summaries of each archive batch.

### 3.4 Future Use Cases

- Process digital legacies of staff who have passed or retired.
- Build living archives for current staff by periodically snapshotting their digital research folders.
- Provide researchers with abstract-level access to request deeper materials.

### 3.5 Next Steps (from Meeting)

- Begin testing the pipeline using a small, non-sensitive sample dataset.
- Prepare a test batch including images and mixed file types.
- Coordinate with institutional data governance to review handling of sensitive and classified data.
- Schedule a follow-up meeting to demo the first prototype and determine the most useful outputs.

---

## 4. Technical Methodology

The following Python-based methods and approaches were explored for redacting sensitive data while maintaining the readability of archived email content.

### 4.1 Word Frequency-Based Redaction (Core Approach)

The primary strategy uses TF-IDF (Term Frequency-Inverse Document Frequency) or simple word frequency analysis to redact infrequently appearing words. This can be improved using stopword filtering and part-of-speech (POS) tagging to retain contextually important words.

**Relevant libraries:** NLTK, spaCy, scikit-learn

### 4.2 Named Entity Recognition (NER) for Sensitive Data Removal

Pre-trained NER models detect and redact sensitive entities including personal names, email addresses, phone numbers, organizations, and locations.

**Relevant libraries:** spaCy, Flair, Hugging Face Transformers

### 4.3 Regex-Based Pattern Matching for PII Redaction

Regular expressions detect structured PII patterns such as email addresses, phone numbers, Social Security Numbers, and credit card numbers.

**Relevant libraries:** re, pandas, numpy

### 4.4 Differential Privacy for Text Redaction

Differential privacy techniques add calibrated noise to word frequencies, reducing the risk of re-identification. Laplace noise injection can be used to obfuscate rare or sensitive words.

**Relevant libraries:** PySyft, DiffPrivLib

### 4.5 Contextual Embeddings for Sensitive Information Detection

Transformer-based models such as BERT, RoBERTa, or GPT understand the contextual meaning of words. Fine-tuned models can detect sensitive words and determine whether they warrant redaction.

**Relevant libraries:** transformers, sentence-transformers, Hugging Face

### 4.6 Clustering and Topic Modeling to Detect Outlier Words

Latent Dirichlet Allocation (LDA) or Non-Negative Matrix Factorization (NMF) detects rare topics in the email dataset, allowing identification and redaction of outlier words that do not belong to frequent topics.

**Relevant libraries:** gensim, sklearn, topic-modeling-toolkit

### 4.7 Rule-Based NLP Redaction with Heuristics

A custom rule-based approach can redact words based on characteristics such as word length (e.g., words over 15 characters), POS tagging (e.g., proper nouns), or sentiment score (e.g., highly negative statements).

**Relevant libraries:** spaCy, textblob, NLTK

### 4.8 Anonymization via Word Replacement

Rather than removing sensitive terms entirely, this approach replaces them with synthetic placeholders (e.g., a personal name becomes a generic identifier, an organization becomes a generic company label). This improves readability while preserving document structure.

**Relevant libraries:** faker, textacy, spacy-anonymizer

### 4.9 Knowledge Graphs for Entity Linking and Redaction

A knowledge graph detects relationships between words and determines which entities require redaction. This is particularly useful for understanding contextual relationships within emails.

**Relevant libraries:** neo4j, networkx, rdflib

### 4.10 Autoencoder-Based Redaction Using Deep Learning

An autoencoder model trained to reconstruct emails can selectively remove sensitive terms. The model learns which words are important for readability and automatically redacts less necessary ones.

**Relevant libraries:** tensorflow, pytorch, keras

---

## 5. Method Comparison and Selection Guide

The appropriate method depends on the primary goal of the redaction task:

| Priority Goal | Recommended Approach(es) |
|---|---|
| **Accuracy** | NER-Based, Contextual Embeddings (BERT/RoBERTa) |
| **Efficiency / Speed** | Regex Matching, TF-IDF Word Frequency, Rule-Based NLP |
| **Privacy Protection** | Differential Privacy, Autoencoder-Based Deep Learning |
| **Readability Preservation** | Word Replacement / Anonymization, Topic Modeling |
| **Scalability** | TF-IDF + NER combined pipeline |
| **Low Resource / Simple Setup** | Regex + Word Frequency (current baseline approach) |

---

## 6. Dataset and Sample Data

The project uses a historical email dataset converted from legacy PST (Personal Storage Table) archive files. The dataset was processed through the following stages:

- **Raw Conversion:** PST files were converted to individual EML files using a custom conversion script.
- **Parsing:** EML files were parsed and structured into a flat CSV format capturing headers, body, and metadata.
- **Cleaning:** The parsed dataset was cleaned to remove formatting artifacts and normalize field values.
- **PII Extraction:** Named entity recognition and regex-based techniques were applied to identify and flag personally identifiable information.
- **Final Dataset:** The cleaned, PII-flagged dataset serves as the primary input for word frequency analysis and redaction experiments.

> **Note:** The raw PST files and original EML archives are not included in this repository due to the presence of real personal correspondence. Only the processed, cleaned dataset is included.

---

## 7. Repository Structure

```
email-archiving-ediscovery-word-frequency/
├── notebooks/
│   ├── Convert_Eml_to_CSV.ipynb
│   └── Data_Table_Preparation_Email_Archiving_Version_1.ipynb
├── data/
│   └── emails_parsed_cleaned_PII_Extracted_v3.csv
├── outputs/
│   ├── Data_Table_Preparation_Email_Archiving_Version_1.html
│   └── Data_Table_Preparation_Email_Archiving_Version_1.pdf
├── docs/
│   └── Email_Archiving_Ediscovery_Project_Documentation.md
├── .gitignore
└── README.md
```

---

## 8. Future Work and Roadmap

- Prototype and benchmark multiple redaction methods against the cleaned dataset.
- Develop an evaluation framework comparing readability scores vs. privacy protection metrics.
- Extend the pipeline to handle non-email file types (PDFs, Word documents, images via OCR).
- Implement semantic search and vectorization layer for archive querying.
- Build a finding aid generator to produce human-readable archive summaries.
- Explore integration with institutional records management systems.
- Evaluate differential privacy approaches for production deployment.

---

*University of Illinois System | Records and Information Management Services | Decision Support*
