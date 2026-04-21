# AI-Powered Email Archiving & E-Discovery Pipeline

## Overview
This project is an end-to-end AI pipeline designed to process, redact, and analyze large volumes of unstructured email data. It enables organizations to transform legacy communication archives into searchable, privacy-aware datasets that are usable for research, compliance, and discovery.

The system focuses on balancing two competing goals:
- preserving readability and context  
- protecting sensitive information  

---

## Problem

Organizations often inherit massive collections of email data from former employees, researchers, or departments. These datasets are:

- unstructured and inconsistent  
- lacking metadata and context  
- filled with sensitive personal or institutional information  
- too large to process manually  

As a result, valuable historical data often remains inaccessible.

---

## Solution

This project introduces a scalable pipeline that:

1. **Ingests** raw email data (PST → EML → structured format)  
2. **Parses and cleans** email content  
3. **Identifies sensitive information (PII)** using multiple techniques  
4. **Applies redaction strategies** while preserving readability  
5. **Generates summaries and structured outputs**  
6. **Enables semantic search** using embeddings and vectorization  

The result is a system that makes email archives:
- searchable  
- analyzable  
- privacy-compliant  

---

## Key Features

- **PII Detection**
  - Named Entity Recognition (NER)
  - Regex-based pattern matching (emails, phone numbers, SSNs)

- **Redaction Strategies**
  - Word frequency filtering (TF-IDF)
  - Context-aware redaction
  - Optional anonymization (replacement vs removal)

- **Semantic Search**
  - Embedding-based retrieval (sentence transformers)
  - Query-driven exploration of archives

- **Scalable Pipeline**
  - Designed to process large, mixed-format datasets
  - Modular architecture for extensibility

---

## Technical Approach

### 1. Word Frequency Redaction
Rare words are more likely to contain identifying information.  
TF-IDF is used to suppress low-frequency terms while preserving core context.

### 2. Named Entity Recognition (NER)
Pre-trained NLP models detect:
- names  
- organizations  
- locations  
- contact information  

### 3. Regex-Based PII Detection
Structured sensitive data is identified using pattern matching:
- email addresses  
- phone numbers  
- IDs  

### 4. Contextual NLP Models
Transformer-based models (BERT, etc.) provide deeper understanding of:
- whether a term is sensitive  
- how critical it is to meaning  

### 5. Semantic Embeddings
Emails are vectorized to enable:
- similarity search  
- topic-based retrieval  
- researcher-friendly querying  

---

## Pipeline Architecture
Raw PST Files
↓
EML Conversion
↓
Parsing & Cleaning
↓
PII Detection (NER + Regex)
↓
Redaction (TF-IDF + Rules)
↓
Summarization
↓
Embedding & Vectorization
↓
Search & Analysis

---

## Tech Stack

- **Python**
- **NLP**: spaCy, NLTK, transformers
- **ML**: scikit-learn (TF-IDF), sentence-transformers
- **Data Processing**: pandas, numpy
- **Optional**:
  - FAISS (vector search)
  - OCR tools for images
  - PyTorch / TensorFlow for advanced models

---

## Dataset

The dataset used in this project originates from historical email archives:

- PST files converted to EML format  
- Parsed into structured CSV format  
- Cleaned and normalized  
- PII flagged using NLP techniques  

> Note: Raw data is not included due to sensitive content.

---

## Results & Impact

This pipeline enables:

- faster archival processing  
- scalable redaction workflows  
- improved accessibility of historical data  
- safer data sharing for research  

It significantly reduces the manual effort required to process large email archives while maintaining privacy standards.

---

## Future Improvements

- Expand support for additional file types (PDF, Word, images via OCR)
- Implement differential privacy techniques
- Improve contextual redaction using fine-tuned LLMs
- Build a UI for interactive search and exploration
- Generate automated “finding aids” for archive collections

---

## Use Cases

- University and institutional archives  
- Legal e-discovery workflows  
- Research data preparation  
- Compliance and data governance  

---

## Author

Tayler Erbe  
Senior Data Scientist  

Focused on applied AI, NLP, and building systems that turn complex data into real-world impact.
