# 📩 Email Data Preprocessing and Message Extraction

**Notebook:** `Email_Archiving_Data_Preparation_v1_CLEAN.ipynb`  
**Dataset:** `emails_parsed_cleaned_PII_Extracted_v3.csv`  
**Project:** Email Archiving and E-Discovery using Word Frequency Analysis  
**Institution:** University of Illinois System — Records and Information Management Services / Decision Support

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Dependencies and Installation](#3-dependencies-and-installation)
4. [Data Description](#4-data-description)
5. [Pipeline Overview](#5-pipeline-overview)
6. [Step-by-Step Notebook Walkthrough](#6-step-by-step-notebook-walkthrough)
   - [Step 1: Load the Parsed Email Data](#step-1-load-the-parsed-email-data)
   - [Step 2: Extract Folder Structure from File Path](#step-2-extract-folder-structure-from-file-path)
   - [Step 3: Extract Email Title](#step-3-extract-email-title)
   - [Step 4: Break Email Chains into Individual Messages](#step-4-break-email-chains-into-individual-messages)
   - [Step 5: Extract Email Header Information](#step-5-extract-email-header-information)
   - [Step 6: Viewing Extracted Email Data](#step-6-viewing-extracted-email-data)
   - [Step 7: Fuzzy Search by Sender](#step-7-fuzzy-search-by-sender)
   - [Step 8: Temporal Analysis — Email Volume Over Time](#step-8-temporal-analysis--email-volume-over-time)
   - [Step 9: Topic Modeling with LDA (Subject Lines)](#step-9-topic-modeling-with-lda-subject-lines)
   - [Step 10: LDA + LLM Topic Labeling (Llama 3.2)](#step-10-lda--llm-topic-labeling-llama-32)
   - [Step 11: PII Detection with Presidio Analyzer](#step-11-pii-detection-with-presidio-analyzer)
   - [Step 12: Scaled PII Extraction Across Full Dataset](#step-12-scaled-pii-extraction-across-full-dataset)
   - [Step 13: LLM-Based Email Summarization (Ollama)](#step-13-llm-based-email-summarization-ollama)
   - [Step 14: Batch Summarization and CSV Output](#step-14-batch-summarization-and-csv-output)
   - [Step 15: Topic Modeling on LLM Summaries](#step-15-topic-modeling-on-llm-summaries)
7. [Output Files](#7-output-files)
8. [Performance Benchmarks](#8-performance-benchmarks)
9. [Privacy and PII Handling](#9-privacy-and-pii-handling)
10. [Optimization Opportunities](#10-optimization-opportunities)
11. [Known Issues and Limitations](#11-known-issues-and-limitations)
12. [Future Work](#12-future-work)

---

## 1. Project Overview

This notebook documents a complete data preprocessing and analysis pipeline for a historical institutional email archive. The emails originate from legacy `.pst` (Personal Storage Table) archive files converted to `.eml` format, representing over 20 years of professional correspondence from a large university facilities and engineering operation.

The primary goals of the pipeline are:

- **Preserve provenance** — Retain the original folder structure, file names, and metadata embedded in the email archive hierarchy.
- **Decompose email threads** — Split multi-message email chains into individual message segments to enable per-message analysis.
- **Extract structured metadata** — Parse sender, recipient, subject, date, and importance fields from email header blocks using regex.
- **Detect and anonymize PII** — Use Microsoft's `presidio-analyzer` library to identify and replace personally identifiable information (names, phone numbers, email addresses, dates, SSNs) with numbered placeholders.
- **Summarize content using a local LLM** — Apply Llama 3.2 via Ollama to rewrite each email as a fully anonymized, human-readable summary suitable for research access.
- **Discover topics** — Apply Latent Dirichlet Allocation (LDA) topic modeling on both subject lines and LLM-generated summaries, and use Llama 3.2 to generate natural language labels for each topic cluster.
- **Enable temporal and sender-based analysis** — Build tools to query email volume over time and filter by sender with fuzzy string matching.

This work supports a broader institutional initiative to make legacy digital archives accessible to researchers without exposing sensitive personal information.

---

## 2. Repository Structure

```
email-archiving-ediscovery-word-frequency/
├── notebooks/
│   ├── Convert_Eml_to_CSV.ipynb                          # PST/EML → CSV conversion
│   └── Email_Archiving_Data_Preparation_v1_CLEAN.ipynb  # This notebook
├── data/
│   └── emails_parsed_cleaned_PII_Extracted_v3.csv       # Sample dataset (anonymized, 10 rows)
├── outputs/
│   ├── Data_Table_Preparation_Email_Archiving_Version_1.html
│   └── Data_Table_Preparation_Email_Archiving_Version_1.pdf
├── docs/
│   └── Email_Archiving_Ediscovery_Project_Documentation.md
├── .gitignore
└── README.md
```

> **Note:** The raw `.pst` and `.eml` archive files are **not included** in this repository. They contain real personal correspondence and are stored securely offline. Only the processed, anonymized sample dataset is included.

---

## 3. Dependencies and Installation

### Python Version
Python 3.8 or higher recommended.

### Required Libraries

```bash
pip install pandas numpy matplotlib scikit-learn nltk fuzzywuzzy python-Levenshtein
pip install presidio-analyzer presidio-anonymizer
pip install spacy
python -m spacy download en_core_web_lg
```

### Optional (for LLM summarization and topic labeling)

Llama 3.2 via Ollama is used for generating natural language topic titles and anonymized email summaries. This runs locally and requires:

1. Install [Ollama](https://ollama.com/) for your operating system.
2. Pull the Llama 3.2 model:
   ```bash
   ollama pull llama3.2
   ```
3. Update the `ollama_path` variable in the notebook to point to your local Ollama executable:
   ```python
   ollama_path = r"C:\Users\[USERNAME]\AppData\Local\Programs\Ollama\ollama.exe"
   ```

### NLTK Data

The notebook downloads the `stopwords` corpus automatically:
```python
import nltk
nltk.download("stopwords")
```

---

## 4. Data Description

### Input: `emails_parsed_cleaned_PII_Extracted_v3.csv`

This is the primary input dataset. It was produced by a preceding notebook (`Convert_Eml_to_CSV.ipynb`) which converted raw `.eml` files into a flat tabular structure. The dataset contains **5,810 rows** in the full version (10 rows in the anonymized sample included in this repo).

Each row represents one email file (or one message segment within a thread, after chain splitting). The 29 columns are:

| Column | Description |
|---|---|
| `Unnamed: 0` | Original row index |
| `file_name` | The `.eml` filename (e.g., `20030610-1444 Subject line.eml`) |
| `file_path` | Full path to the original `.eml` file on the source system |
| `email_text` | The full raw text of the email file, including any chained messages |
| `Folder_1` through `Folder_10` | Path components split on backslash — each level of the archive folder hierarchy |
| `email_title` | The filename without the `.eml` extension (used as a human-readable label) |
| `email_original_message` | One segment of the email chain (populated after chain splitting in Step 4) |
| `from_email` | Parsed sender name/address from the email header |
| `sent_email` | Parsed send date/time string from the email header |
| `to_email` | Parsed recipient(s) from the email header |
| `cc_email` | Parsed CC recipients (if present) |
| `subject_email` | Parsed subject line |
| `importance_email` | Parsed importance flag (e.g., `High`, `Normal`) |
| `email_text_content` | The message body with the header block removed |
| `cleaned_text` | The email text after PII replacement with numbered placeholders |
| `names` | Dict mapping placeholder tokens (e.g., `<PERSON1>`) to the original detected names |
| `phone_numbers` | Dict mapping placeholder tokens to detected phone numbers |
| `email_addresses` | Dict mapping placeholder tokens to detected email addresses |
| `dates` | Dict mapping placeholder tokens to detected date/time expressions |
| `ssn` | Dict mapping placeholder tokens to detected Social Security Numbers |

### Filename Convention

Email filenames follow the pattern `YYYYMMDD-HHMM Subject line.eml`, encoding both the date/time and subject of the message. This makes them sortable chronologically without parsing headers.

---

## 5. Pipeline Overview

The notebook implements a 15-step linear pipeline:

```
Raw CSV Input
    │
    ├── Step 1:  Load CSV
    ├── Step 2:  Extract folder hierarchy from file paths
    ├── Step 3:  Extract email title from filename
    ├── Step 4:  Split email chains → explode to one row per message
    ├── Step 5:  Parse structured headers (From, Sent, To, Cc, Subject, Importance)
    ├── Step 6:  Validate header extraction with sample output
    │
    ├── Step 7:  Fuzzy sender search (fuzzywuzzy)
    ├── Step 8:  Temporal analysis — email volume over time (matplotlib)
    │
    ├── Step 9:  LDA topic modeling on subject lines (sklearn)
    ├── Step 10: LLM topic labeling via Llama 3.2 (Ollama)
    │
    ├── Step 11: PII detection demos (presidio_analyzer)
    ├── Step 12: Scaled PII extraction across full dataset → output CSV
    │
    ├── Step 13: LLM email summarization function (Ollama / Llama 3.2)
    ├── Step 14: Batch summarization → output CSV
    └── Step 15: LDA topic modeling on LLM summaries
```

Each step builds on the previous one. The major transformation checkpoints produce intermediate CSV files that allow the pipeline to be resumed without reprocessing from scratch.

---

## 6. Step-by-Step Notebook Walkthrough

---

### Step 1: Load the Parsed Email Data

**Purpose:** Load the pre-processed email dataset from CSV into a pandas DataFrame.

```python
import os
import pandas as pd

df = pd.read_csv(r"data/emails_parsed_cleaned_PII_Extracted_v3.csv")
df.head()
```

The dataset at this stage contains the raw email text alongside any previously extracted PII fields. The `file_path` column is the key anchor — it contains the full path to the original `.eml` file and encodes the archive folder structure.

---

### Step 2: Extract Folder Structure from File Path

**Purpose:** The file path encodes the entire archive folder hierarchy. Splitting it on backslashes creates individual columns (`Folder_1` through `Folder_N`) that represent each level of nesting — PST file name, folder name, subfolder name, and ultimately the filename itself.

```python
split_path_df = df["file_path"].str.split(r'\\', expand=True)
split_path_df.columns = [f"Folder_{i+1}" for i in range(split_path_df.shape[1])]
df = df.join(split_path_df)
```

**Why this matters:** The folder names in the original archive were meaningful — they were organized by project, system, department, and topic (e.g., `APP- Admin- Safety`, `APP- Sys- Water Treatment Genl`, `APP- Finance- Budget Reps`). Preserving this hierarchy as structured columns makes it possible to filter, group, and analyze emails by their original organizational context without relying solely on free-text search.

**Example folder hierarchy extracted:**

| Folder_1 | Folder_2 | Folder_3 | Folder_4 | Folder_5 | Folder_6 | Folder_7 |
|---|---|---|---|---|---|---|
| C: | Emails_Archiving | Archive_Converted | archive1.pst | Archive | Top of Personal Folders | Finance- Budget |

---

### Step 3: Extract Email Title

**Purpose:** Create a clean, human-readable label for each email by extracting the filename without its `.eml` extension and storing it in a new `email_title` column.

```python
def get_email_title(path):
    if not path:
        return ""
    base_name = os.path.basename(path)
    title_without_ext, _ = os.path.splitext(base_name)
    return title_without_ext

df["email_title"] = df["file_path"].apply(get_email_title)
df = df.reset_index()
```

**Example:** `20030610-1444 Update on project status.eml` → `20030610-1444 Update on project status`

This title is stable and unique within the archive, making it useful as a display label and as a join key when cross-referencing between the different intermediate CSV outputs produced by the pipeline.

---

### Step 4: Break Email Chains into Individual Messages

**Purpose:** Many `.eml` files contain entire conversation threads — a reply that includes the previous message, which itself includes an earlier message, and so on. The full `email_text` field captures all of this. To analyze each individual message in isolation (for PII extraction, summarization, and header parsing), we need to split the thread into its component messages.

**Approach:** The string `-----Original Message-----` is a standard delimiter that email clients insert when a message is forwarded or replied to. We split on this delimiter and then use pandas `.explode()` to give each segment its own row.

```python
df["email_original_message"] = df["email_text"].str.split("-----Original Message-----")
df_exploded = df.explode("email_original_message").reset_index(drop=True)
df_exploded["email_original_message"] = df_exploded["email_original_message"].str.strip()
```

**Scale impact:** This step significantly increases the number of rows. In the full dataset, 5,810 original email files expanded to approximately 79,676 individual message segments — roughly 13–14 message segments per file on average — reflecting the depth of the conversation threads present in the archive.

**Important:** The original `email_text` column (the full thread) is retained alongside `email_original_message` (the individual segment). This means downstream analysis can access either the full conversation context or the isolated message.

---

### Step 5: Extract Email Header Information

**Purpose:** When a message is part of a chain, the forwarded segments include a structured header block at the top (From, Sent, To, Cc, Subject, Importance). This step parses those fields out of each message using a regex pattern, creating dedicated columns for each header field and separating the header from the message body.

**The `extract_headers()` function:**

```python
def extract_headers(message):
    header_pattern = (
        r"^From:\s*(?P<from_email>.*?)\r?\n"
        r"Sent:\s*(?P<sent_email>.*?)\r?\n"
        r"To:\s*(?P<to_email>.*?)\r?\n"
        r"(?:Cc:\s*(?P<cc_email>.*?)\r?\n)?"       # optional
        r"Subject:\s*(?P<subject_email>.*?)\r?\n"
        r"(?:Importance:\s*(?P<importance_email>.*?)\r?\n)?"  # optional
        r"\r?\n"  # blank line marks end of header block
    )
    m = re.search(header_pattern, message, re.DOTALL)
    if m:
        headers = m.groupdict()
        headers["email_text_content"] = message[m.end():].lstrip()
        return headers
    else:
        return {field: None for field in [...], "email_text_content": message}
```

**Key design decisions:**
- `Cc` and `Importance` are marked as optional using `(?:...)?` because many messages omit them.
- The function returns `None` for all header fields (not an empty string) when no header block is detected, allowing downstream code to distinguish between "no header found" and "header present but field was empty."
- The remainder of the message after the header block is stored in `email_text_content` — this is the clean message body used in all downstream processing.

**Output columns added:** `from_email`, `sent_email`, `to_email`, `cc_email`, `subject_email`, `importance_email`, `email_text_content`

---

### Step 6: Viewing Extracted Email Data

**Purpose:** Validation step. After header extraction, we print a sample record to confirm that the `from_email`, `sent_email`, `to_email`, `cc_email`, `subject_email`, and `importance_email` fields have been correctly parsed, and that `email_text_content` contains only the message body without the header block.

```python
print("Original Email:\n")
print(df_exploded["email_original_message"][sample_row])
print("\nExtracted Header Contents:\n")
print("From: " + str(df_exploded["from_email"][sample_row]))
# ... etc.
```

This check is important because the regex pattern depends on consistent header formatting, which can vary across email clients and time periods. If the pattern fails to match, the function gracefully falls back to storing the entire message in `email_text_content` with all header fields set to `None`.

---

### Step 7: Fuzzy Search by Sender

**Purpose:** Enable filtering of emails by sender even when the `from_email` field contains inconsistent name formatting. In older email archives, the same person might appear as `"Smith, Jane"`, `"Jane Smith (Dept)"`, `"J. Smith <jsmith@domain.edu>"`, or variations thereof.

**Approach:** We use `fuzzywuzzy`'s `partial_ratio` function, which returns a similarity score (0–100) between two strings. A threshold of 80 catches most reasonable name variations without generating too many false positives.

```python
from fuzzywuzzy import fuzz

target_name = "Sender, Example"
threshold = 80

matches = df_exploded["from_email"].apply(
    lambda x: fuzz.partial_ratio(str(x).lower(), target_name.lower()) >= threshold
)
filtered_emails = df_exploded[matches]
```

**Why `partial_ratio` over `ratio`?** `partial_ratio` computes the similarity of the best-matching substring rather than the full string, making it much better at matching short names against longer formatted strings like `"Smith, Jane A. (Engineering)"`.

**Result in full dataset:** Searching for a single sender returned 386 matching rows out of ~79,000, demonstrating that even in a large archive a targeted sender search is highly specific.

---

### Step 8: Temporal Analysis — Email Volume Over Time

**Purpose:** Understand communication patterns across time — how active was the archive overall, and how did activity from specific senders evolve over the years?

**Steps:**
1. Parse the `sent_email` string field to a proper datetime using `pd.to_datetime(..., errors="coerce")`. The `errors="coerce"` argument turns unparseable dates into `NaT` rather than raising an error, which is important given the inconsistent date formats across a 20+ year archive.
2. Create a `year_month` period column using `.dt.to_period("M")` for monthly aggregation.
3. Count emails per period and plot as a horizontal bar chart (full archive) or vertical bar chart (single sender).

```python
df_exploded["sent_email_parsed"] = pd.to_datetime(df_exploded["sent_email"], errors="coerce")
df_exploded["year_month"] = df_exploded["sent_email_parsed"].dt.to_period("M")
monthly_counts = df_exploded["year_month"].value_counts().sort_index()
```

**Visualization:** The full-archive chart uses a horizontal bar layout because the date range spans many years and vertical labels would overlap. The single-sender chart uses a vertical bar layout since the active period is typically a shorter window.

---

### Step 9: Topic Modeling with LDA (Subject Lines)

**Purpose:** Automatically discover the major themes or topic areas present in the archive by analyzing the subject lines of the emails. This gives a high-level content map of the archive without reading individual messages.

**Process:**

1. **Text cleaning** — Subject lines are lowercased, stripped of digits and non-word characters, and filtered to remove English stopwords.
2. **Vectorization** — `CountVectorizer` converts the cleaned subject lines into a document-term matrix (bag-of-words representation). Parameters `max_df=0.9` (ignore terms in >90% of documents, i.e., near-universal words) and `min_df=2` (ignore terms appearing in fewer than 2 documents, i.e., rare one-offs) are used to focus on meaningful mid-frequency vocabulary.
3. **LDA** — `LatentDirichletAllocation` with `n_components=5` is fitted on the document-term matrix. LDA is an unsupervised probabilistic model that assumes each document is a mixture of topics and each topic is a distribution over words.
4. **Topic assignment** — Each email subject is assigned to its dominant topic using `np.argmax()` on the topic probability matrix.
5. **Visualization** — A bar chart shows the number of email subjects assigned to each topic number.
6. **Keyword display** — The top 10 keywords for each topic are printed by sorting the LDA component weights.

```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.decomposition import LatentDirichletAllocation

vectorizer = CountVectorizer(max_df=0.9, min_df=2, stop_words="english")
doc_term_matrix = vectorizer.fit_transform(df_exploded["subject_clean"])
lda_model = LatentDirichletAllocation(n_components=5, random_state=42)
lda_topics = lda_model.fit_transform(doc_term_matrix)
```

**Limitation of LDA alone:** The keyword lists produced by LDA are often ambiguous or difficult to interpret without domain knowledge. A list like `['fw', 'ok', 'rdm', 'boiler', 'water', 'xa', 'po', 'outage', 'status', 'notification']` is meaningful to someone familiar with the archive but opaque to an outside researcher. This motivates Step 10.

---

### Step 10: LDA + LLM Topic Labeling (Llama 3.2)

**Purpose:** Convert the abstract keyword clusters produced by LDA into short, human-readable topic titles by passing the top keywords to a local large language model.

**Approach:** The `generate_topic_title()` function sends a prompt to Llama 3.2 via the Ollama CLI using `subprocess.run()`. The prompt asks the model to act as a topic modeling expert and infer a descriptive 5-word-or-fewer title from the keyword list.

```python
def generate_topic_title(keywords):
    prompt = f"""
You are an expert at topic modeling and naming. Based on the following list of keywords,
provide a short descriptive title (5 words or fewer) for this topic.
Keywords: {', '.join(keywords)}
"""
    result = subprocess.run([ollama_path, "run", model_name, "--", prompt], ...)
    return result.stdout.strip()
```

**Why run locally?** The email content, even after cleaning, may contain sensitive context. Running the LLM locally via Ollama ensures no data is sent to external APIs.

**Result:** LDA keyword clusters like `['steam', 'gas', 'plant', 'fw', 'maintenance', 'visit', 'controls', 'tunnel', 'service']` become labeled topics like `"Steam Plant Operations and Controls"` — a label that is immediately meaningful to archivists, researchers, and administrators without any domain knowledge.

The Llama-generated titles are then used to label the x-axis of the topic distribution bar chart, producing a visualization that communicates content themes clearly.

---

### Step 11: PII Detection with Presidio Analyzer

**Purpose:** Demonstrate the capabilities of Microsoft's `presidio-analyzer` library before applying it at scale. This section works through five different approaches to handling detected PII, from simple replacement to blackout redaction to cryptographic hashing.

**What `AnalyzerEngine` detects** (among others):
- `PERSON` — Personal names
- `PHONE_NUMBER` — Phone numbers in various formats
- `EMAIL_ADDRESS` — Email addresses
- `DATE_TIME` — Dates and times
- `US_SSN` — US Social Security Numbers
- `CREDIT_CARD` — Credit card numbers
- `IP_ADDRESS` — IP addresses
- `UK_NHS` — UK NHS numbers (can misfire on US phone numbers — see Known Issues)

**The five anonymization approaches demonstrated:**

#### Approach 1: Replace with Entity Type Placeholders
```python
# Input:  "My name is John Doe and my number is 312-555-1234."
# Output: "My name is <PERSON> and my number is <UK_NHS>."
anonymizer.anonymize(text=text, analyzer_results=results)
```

#### Approach 2: Full Redaction (Remove PII Entirely)
```python
# Output: "My name is  and my number is . I was born on ."
operators = {"DEFAULT": OperatorConfig("redact")}
anonymizer.anonymize(text=text, analyzer_results=results, operators=operators)
```

#### Approach 3: Blackout Style (Replace with Block Characters)
```python
# Output: "My name is ████████ and my number is ████████████. I was born on ██████████."
def custom_blackout(text, results):
    # Replaces each detected span with █ characters matching the original length
```
This preserves the visual length of the redacted content, which can help readers understand the structure of the original text even after anonymization.

#### Approach 4: Partial Masking (Last 4 Digits of Phone Numbers)
```python
# Output: "My name is John and my number is ***-***-1234."
def custom_mask_phone(text, results):
    # Replaces all but the last 4 digits with asterisks, preserving formatting characters
```
Useful for audit or compliance use cases where the last 4 digits are needed for identification but full numbers must not be stored.

#### Approach 5: Cryptographic Hashing (Names → SHA-256)
```python
# Output: "My name is 6cea57c2fb6cbc2a40411135... and I work with a2dd3acadb1..."
operators = {"PERSON": OperatorConfig("hash"), "EMAIL_ADDRESS": OperatorConfig("redact")}
```
Hashing is useful when you need to track communication patterns (e.g., "did Person A ever communicate with Person B?") without storing the actual names. The same name always produces the same hash, so co-occurrence analysis is still possible.

---

### Step 12: Scaled PII Extraction Across Full Dataset

**Purpose:** Apply the `clean_text_with_presidio()` function systematically to every row in the dataset, producing a new CSV where each email's `email_text_content` has been replaced with a `cleaned_text` version containing numbered placeholders, and the original PII values are stored in separate mapping dictionary columns.

**The `clean_text_with_presidio()` function:**

The function maps five Presidio entity types to five output columns:

```python
pii_mapping = {
    "PERSON":                    "names",
    "PHONE_NUMBER":              "phone_numbers",
    "EMAIL_ADDRESS":             "email_addresses",
    "DATE_TIME":                 "dates",
    "US_SOCIAL_SECURITY_NUMBER": "ssn"
}
```

For each detected entity, the function:
1. Assigns a numbered placeholder (`<PERSON1>`, `<PERSON2>`, etc.)
2. Stores the mapping `{"<PERSON1>": "original_value"}` in the appropriate column
3. Replaces the span in the text with the placeholder

Processing is done **in reverse order by position** (right to left through the string) to prevent character offset drift as replacements are made.

**Batch processing with resume support:**

The pipeline processes rows in batches of 10 and appends each batch to the output CSV immediately. If the process is interrupted (which is likely given the scale — see Performance section), it resumes from where it left off by checking how many rows have already been written to the output file.

```python
start_index = pd.read_csv(output_csv).shape[0] if os.path.exists(output_csv) else 0

for i in range(start_index, len(df_exploded), batch_size):
    batch = df_exploded.iloc[i:i+batch_size].copy()
    # ... process and append ...
```

**Output:** `emails_parsed_cleaned_PII_Extracted_v3.csv` — the primary cleaned dataset containing 5,810 rows × 29 columns.

---

### Step 13: LLM-Based Email Summarization (Ollama)

**Purpose:** Even after PII extraction, the cleaned text still contains content that may be sensitive or confusing out of context — references to specific projects, systems, or events that could inadvertently identify individuals or reveal confidential operational information. This step uses a local LLM to rewrite each email as a short, generic, fully anonymized summary that captures the intent and topic of the message without any specific identifying details.

**The `rewrite_email_summary()` function:**

```python
def rewrite_email_summary(email_text):
    prompt = f"""
Rewrite the following email as a concise summary that omits or generalizes all
sensitive or personal details. Replace names with "the writer" or "a contact",
remove specific dates, phone numbers, and locations.

Email text:
{email_text}
"""
    result = subprocess.run([ollama_path, "run", model_name, "--", prompt], ...)
    return result.stdout.strip()
```

**Two-stage anonymization:** This step works best when applied to the **already PII-extracted** text (i.e., text where names have already been replaced with `<PERSON1>` etc.). The LLM then has less identifying information to work with and can focus on summarizing the content rather than detecting PII.

**Example transformation:**

Input (after PII extraction):
```
<PERSON6>,
Below is <PERSON5>'s response to my request to meet with them. They referred me
to <PERSON4>, but it looks good so far. Do I need to notify anyone else?
<PERSON1>
```

Output (LLM summary):
```
A meeting with a contact has been scheduled, and another individual has been
informed of the arrangement. The writer is unclear whether additional parties
need to be notified about the upcoming event.
```

The LLM summary is appropriate for research access — it communicates the nature of the communication (scheduling coordination, seeking approval) without revealing who was involved or what specific project was being discussed.

---

### Step 14: Batch Summarization and CSV Output

**Purpose:** Apply `rewrite_email_summary()` across the full cleaned dataset in batches, with resume support identical to Step 12.

```python
for i in range(start_index, len(df), batch_size):
    batch = df.iloc[i:i+batch_size].copy()
    for index, row in batch.iterrows():
        summary = rewrite_email_summary(row["cleaned_text"])
        batch.at[index, "email_summaries"] = summary
    batch.to_csv(output_csv, index=False, mode='a', header=(i == 0 and not os.path.exists(output_csv)))
```

**Output:** `emails_parsed_rewritten_summaries.csv` — contains all columns from the PII-extracted dataset plus the new `email_summaries` column. In the processed sample, 1,630 rows were completed.

**Performance note:** This is the slowest step in the pipeline. See the Performance section for detailed timing estimates.

---

### Step 15: Topic Modeling on LLM Summaries

**Purpose:** Repeat the LDA + Llama topic labeling pipeline from Steps 9–10, but this time running on the LLM-generated summaries (`email_summaries` column) rather than the raw subject lines. This produces a second, often richer view of the archive's content themes.

**Why run topic modeling on summaries instead of (or in addition to) subject lines?**

- Subject lines are short and can be cryptic (`"RE: FW: RE: update"`, `"Question"`, `"FYI"`)
- LLM summaries are longer, more consistently worded, and already stripped of PII and jargon
- Topic modeling on summaries tends to produce more semantically coherent and interpretable clusters

**Configuration change:** For the summary-based run, `num_topics` is increased from 5 to 10 to capture the greater thematic diversity present in the longer summary texts.

**Example topics discovered from summaries (vs. subject lines):**

| Topic # | LDA Keywords | Llama Title |
|---|---|---|
| 0 | contact, work, return, individual, status, regarding | "Record Access and Status Requests" |
| 1 | plant, writer, operator, needs, information, schedule | "Plant Operation Management Guidelines" |
| 2 | safety, records, location, record, retention, osha | "OSHA Safety Record Keeping" |
| 3 | access, project, contact, location, request, files | "Project File Access Requests" |
| 4 | employee, meeting, issue, writer, location, time | "Employee Issue Resolution" |
| 7 | employee, contact, resources, relations, related, work | "Employee Relations and HR" |

---

## 7. Output Files

| File | Description | Rows | Columns |
|---|---|---|---|
| `emails_parsed_exploded.csv` | After chain splitting and header extraction | ~79,676 | 22 |
| `emails_parsed_cleaned_PII_Extracted_v3.csv` | After PII extraction | 5,810 | 29 |
| `emails_parsed_rewritten_summaries.csv` | After LLM summarization | 1,630 (partial) | 30 |

> The sample dataset included in this repository (`data/emails_parsed_cleaned_PII_Extracted_v3.csv`) contains 10 rows of fully synthetic dummy data that mirrors the column structure and format of the real dataset.

---

## 8. Performance Benchmarks

All timing measurements were taken on a standard consumer laptop running the pipeline sequentially.

### PII Extraction (Presidio)

| Batch (10 rows) | Avg Per Row | 100K Rows | 1M Rows |
|---|---|---|---|
| 10 sec | 1.0 sec | ~27.8 hrs | ~278 hrs |
| 12 sec | 1.2 sec | ~33.3 hrs | ~333 hrs |
| 15 sec | 1.5 sec | ~41.7 hrs | ~417 hrs |
| 20 sec | 2.0 sec | ~55.6 hrs | ~556 hrs |

### LLM Summarization (Ollama / Llama 3.2)

| Observed (10 rows) | Avg Per Row | 100K Rows | 1M Rows |
|---|---|---|---|
| 347.80 sec | ~34.78 sec | ~966 hrs (~40 days) | ~9,667 hrs (~403 days) |

**Current bottleneck:** The Ollama CLI invocation (`subprocess.run()`) loads the model from disk on every call. For 5,810 rows, this represents thousands of model-load operations that dwarf the actual inference time.

---

## 9. Privacy and PII Handling

This project was designed with privacy protection as a first-class concern throughout.

**What is NOT included in this repository:**
- Raw `.pst` or `.eml` archive files
- Any CSV containing real names, email addresses, or correspondence content
- Any intermediate processing files from the original pipeline run

**What IS included:**
- Fully synthetic dummy data (10 rows) that mirrors the column structure
- Code that demonstrates the PII extraction and anonymization pipeline
- Anonymized example outputs in notebook markdown cells

**Two-layer anonymization approach:**
1. **Presidio** handles structured PII (names, phone numbers, emails, dates, SSNs) with high recall using NLP models and regex.
2. **LLM summarization** handles unstructured PII and context-dependent sensitive content that rule-based approaches may miss.

**PII mapping preservation:** The mapping dictionaries stored in the `names`, `phone_numbers`, etc. columns allow the pipeline to be audited or reversed if needed, but these mappings must themselves be treated as sensitive data and stored securely.

---

## 10. Optimization Opportunities

| Approach | Expected Impact | Complexity |
|---|---|---|
| **Parallelization** — `multiprocessing` or `joblib` for PII extraction | 4–8× speedup on multi-core systems | Medium |
| **Persistent LLM Runtime** — Run Ollama in server mode (`ollama serve`) and use the REST API instead of CLI | Eliminates model reload overhead; ~10× speedup | Medium |
| **Prompt Simplification** — Shorter summarization prompts reduce token count and inference time | 20–40% speedup | Low |
| **GPU Acceleration** — Run Ollama on a GPU | 10–50× speedup over CPU depending on GPU | Low (hardware) |
| **Cloud API Summarization** — Replace Ollama with Claude, OpenAI, or Gemini API | Near-instant inference; trade-off: data leaves local environment | Low |
| **Persistent `AnalyzerEngine`** — Instantiate once outside the per-row function | Eliminates repeated model loading for Presidio | Low |
| **Incremental Caching** — Hash row IDs to skip already-processed rows | Enables safe parallel runs and crash recovery | Medium |
| **Batch Inference** — Send multiple emails in one LLM prompt | Reduces per-row overhead | Medium |

---

## 11. Known Issues and Limitations

**UK_NHS false positives:** Presidio's `UK_NHS` entity recognizer fires on some US phone numbers that happen to match the NHS number format. In practice this means some phone numbers get labeled as `UK_NHS` rather than `PHONE_NUMBER`. This does not affect the anonymization outcome (the entity is still masked) but does affect the column it's stored in.

**Date over-detection:** The `DATE_TIME` recognizer is aggressive and will flag relative time expressions like "today", "this week", "the next several weeks", and "yesterday afternoon" as dates. For archival purposes this is generally desirable behavior, but it can produce noisy `dates` dictionaries with many entries.

**Email chain splitting:** The `-----Original Message-----` delimiter is not universal. Some email clients use different delimiters (`From:` followed by a date, `________`, `>` quoted text, etc.). Messages from clients that use different formatting conventions will not be split and will remain as single-segment rows with all chain content in one `email_original_message` entry.

**Header regex fragility:** The header extraction regex assumes a specific field order (From → Sent → To → Cc → Subject → Importance). Emails with headers in different orders, or with additional fields between the standard ones, will fail to match and fall through to the "no header found" fallback.

**LLM non-determinism:** Llama 3.2 summaries are not fully deterministic. Re-running the pipeline on the same input may produce slightly different summary text, which means results are not perfectly reproducible without fixed random seeds (which Ollama does not currently expose via the CLI).

**Sequential processing:** The current implementation processes rows one at a time within batches. True parallel processing would require refactoring to use Python's `multiprocessing` module or an async task queue.

---

## 12. Future Work

- **Evaluation framework** — Develop quantitative metrics for anonymization quality (recall of PII detection) and summary quality (semantic similarity to original, readability scores).
- **Extended file type support** — Apply the same pipeline to non-email file types in the archive (Word documents, PDFs, scanned images via OCR, spreadsheets).
- **Semantic search layer** — Embed all summaries using a sentence transformer model and build a vector index (e.g., FAISS or ChromaDB) to enable semantic query-based retrieval of archive content.
- **Finding aid generator** — Automatically produce structured finding aid documents (in EAD XML or plain text) describing the contents of each archive batch, suitable for submission to institutional repositories.
- **Records management integration** — Connect the pipeline output to institutional records management systems for automated retention scheduling and disposition tracking.
- **Interactive UI** — Build a simple web interface allowing archivists to search, filter, and review anonymized summaries without needing to run the notebook directly.
- **Differential privacy** — Explore applying differential privacy techniques to the word frequency outputs to provide formal privacy guarantees beyond heuristic PII masking.

---

*University of Illinois System | Records and Information Management Services | Decision Support*
