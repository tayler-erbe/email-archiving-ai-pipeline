# 📂 Convert EML Files to CSV

**Notebook:** `Convert_Eml_to_CSV_CLEAN.ipynb`  
**Pipeline Stage:** Stage 1 of 2  
**Project:** Email Archiving and E-Discovery using Word Frequency Analysis  
**Institution:** University of Illinois System — Records and Information Management Services / Decision Support

---

## Table of Contents

1. [Overview](#1-overview)
2. [Where This Fits in the Pipeline](#2-where-this-fits-in-the-pipeline)
3. [Prerequisites](#3-prerequisites)
4. [Input](#4-input)
5. [Output](#5-output)
6. [Step-by-Step Walkthrough](#6-step-by-step-walkthrough)
   - [Step 1: Import Libraries](#step-1-import-libraries)
   - [Step 2: The `extract_email_text()` Function](#step-2-the-extract_email_text-function)
   - [Step 3: Configure Paths](#step-3-configure-paths)
   - [Step 4: Walk the Archive and Extract Content](#step-4-walk-the-archive-and-extract-content)
   - [Step 5: Save to CSV](#step-5-save-to-csv)
   - [Step 6: Spot-Check the Output](#step-6-spot-check-the-output)
7. [Understanding EML and MIME](#7-understanding-eml-and-mime)
8. [Design Decisions and Rationale](#8-design-decisions-and-rationale)
9. [What This Notebook Does NOT Capture](#9-what-this-notebook-does-not-capture)
10. [Known Issues and Edge Cases](#10-known-issues-and-edge-cases)
11. [Troubleshooting](#11-troubleshooting)
12. [Next Step](#12-next-step)

---

## 1. Overview

This notebook is the **first stage** of the email archiving and e-discovery pipeline. Its sole responsibility is to convert a folder of raw `.eml` email files into a single, flat CSV file that downstream notebooks can load and process.

The input is a directory tree of `.eml` files — individual email messages exported from legacy `.pst` (Personal Storage Table) archive files using an external conversion tool. The output is a CSV with one row per email, containing three fields: the filename, the full file path, and the extracted plain-text body content.

This step is deliberately narrow in scope. It does not parse email headers, split email chains, detect PII, or perform any analysis. It simply reads every `.eml` file in the archive, extracts its readable text content, and writes everything to a single structured file. All further processing is handled by the downstream notebook.

**Scale:** When run against the full institutional archive, this notebook processed **44,677 individual `.eml` files** across a deeply nested PST folder hierarchy.

---

## 2. Where This Fits in the Pipeline

```
Legacy .pst files
        │
        │  (converted externally to .eml files)
        ▼
Folder of .eml files
        │
        │  ◄── THIS NOTEBOOK
        ▼
emails_parsed.csv                  (one row per .eml file, 3 columns)
        │
        │  (downstream notebook)
        ▼
emails_parsed_exploded.csv         (one row per message segment, 22 columns)
        │
        ▼
emails_parsed_cleaned_PII_Extracted_v3.csv    (PII anonymized, 29 columns)
        │
        ▼
emails_parsed_rewritten_summaries.csv         (LLM summaries added, 30 columns)
```

This notebook produces the very first structured artifact in the pipeline. Everything that follows depends on the CSV it generates.

---

## 3. Prerequisites

### Python version
Python 3.8 or higher.

### Required libraries
All three libraries used in this notebook are part of the Python standard library or are widely available:

```bash
pip install pandas
```

`os` and `email` are part of Python's standard library and require no installation.

### Input data
You need a directory of `.eml` files. These are typically produced by exporting a `.pst` file (Microsoft Outlook's Personal Storage Table format) using a conversion tool. Common tools include:

- **readpst** (Linux/macOS, open source) — converts `.pst` to individual `.eml` files while preserving folder structure
- **Aid4Mail** (Windows, commercial)
- **Kernel OST to PST Converter** and similar utilities

The notebook is designed to work with **any** directory of `.eml` files regardless of how they were created. The folder structure nesting depth does not matter — the script walks the full tree recursively.

### Output directory
The directory you specify for the output CSV must either already exist or you must have permission to create it. The notebook handles directory creation automatically using `os.makedirs(..., exist_ok=True)`.

---

## 4. Input

### Source format: `.eml` files

`.eml` is a plain-text file format for individual email messages, defined by [RFC 2822](https://datatracker.ietf.org/doc/html/rfc2822). Each `.eml` file contains a complete email: headers at the top (From, To, Date, Subject, MIME-Version, Content-Type, etc.) followed by the message body, which may itself be structured as a MIME multipart message.

### Directory structure

The notebook does not impose any requirements on how the `.eml` files are organized within the input directory. It uses `os.walk()` to traverse the full tree recursively, so files at any depth of nesting are found and processed.

In the institutional archive that motivated this work, the directory structure mirrored the original Outlook folder hierarchy from the PST files — for example:

```
Archive_Converted/
├── 03Active1.pst/
│   └── 03 Active/
│       └── Top of Personal Folders/
│           ├── APP- Admin- Safety/
│           │   ├── 20020301-0900 Safety inspection.eml
│           │   └── 20020315-1430 RE- Safety inspection.eml
│           ├── APP- Finance- Budget/
│           │   └── 20010601-0800 Q1 budget review.eml
│           └── ...
├── 1Admin.pst/
│   └── ...
└── repaired.pst/
    └── ...
```

This folder structure encodes valuable metadata about the organizational context of each email (which project, department, or topic area it belonged to). Preserving the full file path in the output CSV allows downstream notebooks to reconstruct and query this hierarchy.

### Filename convention

In the archive used for this project, email filenames follow the pattern:

```
YYYYMMDD-HHMM Subject line of email.eml
```

For example: `20030610-1444 Update on project status.eml`

This convention encodes both the date/time and subject of the original message directly in the filename, making the files chronologically sortable by name without needing to parse headers. This naming pattern appears to have been applied automatically by the PST-to-EML conversion tool.

---

## 5. Output

### File: `emails_parsed.csv`

A flat CSV file with one row per `.eml` file processed. It contains exactly three columns:

| Column | Type | Description |
|---|---|---|
| `file_name` | string | The `.eml` filename only (no directory path). Example: `20030610-1444 Update on project status.eml` |
| `file_path` | string | The full absolute path to the source `.eml` file, including every level of the folder hierarchy. Example: `C:\Emails_Archiving\Archive_Converted\03Active1.pst\03 Active\Top of Personal Folders\APP- Admin- Safety\20030610-1444 Update on project status.eml` |
| `email_text` | string | The extracted plain-text body content of the email. For email chains (replies/forwards), this contains the full thread as a single string. Empty string if the email contained no readable text content. |

### Scale

On the full institutional archive, this CSV contains **44,677 rows** — one per `.eml` file. File size will vary depending on email content length, but typically ranges from tens to hundreds of megabytes for an archive of this scale.

### Important note on `email_text` for chained messages

For `.eml` files that contain forwarded or replied-to messages (i.e., the full conversation thread is embedded in a single file), the `email_text` column will contain the **entire thread as one block of text**. The individual messages within the thread are separated by the string `-----Original Message-----`.

Splitting this block into individual per-message rows is handled in the downstream notebook (`Email_Archiving_Data_Preparation_v1_CLEAN.ipynb`), not here.

---

## 6. Step-by-Step Walkthrough

---

### Step 1: Import Libraries

```python
import os
import email
import pandas as pd
```

Three libraries, each with a distinct role:

**`os`** handles everything related to the file system. Specifically, `os.walk()` traverses the directory tree and `os.path.join()` constructs platform-correct file paths. Using `os.path.join()` rather than string concatenation ensures the code works correctly on both Windows (backslash separators) and Unix/macOS (forward slash separators).

**`email`** is Python's built-in library for parsing RFC 2822 email messages. It handles the complexity of MIME (Multipurpose Internet Mail Extensions) encoding — the standard that allows emails to have multiple content types (plain text, HTML, images, attachments) bundled together in a single file. The `email.message_from_file()` function reads a file handle and returns a `Message` object representing the parsed email structure.

**`pandas`** is used at the end of the pipeline to convert the list of extracted records into a DataFrame and write it to CSV. It is not used during the extraction itself — results are accumulated in a plain Python list for performance, and only converted to a DataFrame once all files have been processed.

---

### Step 2: The `extract_email_text()` Function

This is the core of the notebook. The function takes a single file path, opens the `.eml` file, parses it, and returns the plain-text body as a string.

```python
def extract_email_text(file_path):
    text_content = ""
    try:
        with open(file_path, 'r', encoding='utf-8', errors='replace') as f:
            msg = email.message_from_file(f)
    except Exception as e:
        print(f"Error reading {file_path}: {e}")
        return text_content
    ...
```

#### Opening the file

The file is opened in text mode (`'r'`) with UTF-8 encoding and `errors='replace'`. This means any byte sequence that is not valid UTF-8 is replaced with the Unicode replacement character (`\ufffd`) rather than raising a `UnicodeDecodeError`. This is essential for a 20+ year archive where emails were written by many different email clients across many different operating systems, using many different character encodings. Some older emails may use Windows-1252, ISO-8859-1, or other legacy encodings that are partially incompatible with UTF-8.

If the file cannot be opened at all (e.g., due to a permissions error or a corrupted file), the exception is caught, a message is printed, and the function returns an empty string. This allows the pipeline to continue processing the rest of the archive even if individual files fail.

#### The multipart case

Most modern emails are MIME multipart messages — they contain both a `text/plain` version and a `text/html` version of the same content (to support email clients that can display HTML as well as those that cannot). They may also include attachments as additional MIME parts.

```python
if msg.is_multipart():
    for part in msg.walk():
        if part.get_content_type() == "text/plain" and not part.get('Content-Disposition'):
            try:
                charset = part.get_content_charset() or 'utf-8'
                part_text = part.get_payload(decode=True).decode(charset, errors='replace')
                text_content += part_text + "\n"
            except Exception as e:
                print(f"Error decoding part of {file_path}: {e}")
```

`msg.walk()` traverses every MIME part in the message, including nested multipart containers. The filter `part.get_content_type() == "text/plain"` selects only the plain-text parts and skips HTML, images, and other content types. The additional filter `not part.get('Content-Disposition')` excludes any `text/plain` parts that are marked as file attachments — for example, a `.txt` file attached to the email would have `Content-Disposition: attachment`, and we want to skip it.

The `charset` is read from the MIME part's own headers (where it is declared by the original email client). If no charset is declared, we fall back to UTF-8. The `get_payload(decode=True)` call handles base64 and quoted-printable transfer encodings, returning raw bytes that we then decode with the charset.

Each text part is concatenated with a newline separator, so if an email legitimately has multiple `text/plain` parts they are all captured.

#### The single-part case

Older emails, particularly those written before HTML email became standard, are often simple single-part messages containing only plain text.

```python
else:
    try:
        payload = msg.get_payload(decode=True)
        if payload:
            charset = msg.get_content_charset() or 'utf-8'
            text_content = payload.decode(charset, errors='replace')
        else:
            text_content = ""
    except Exception as e:
        print(f"Error decoding {file_path}: {e}")
```

`get_payload(decode=True)` returns `None` if there is no payload (e.g., the email body is empty), so we check for this before decoding. An empty email body results in an empty string in the output.

#### Return value

```python
return text_content.strip()
```

The `.strip()` call removes any leading or trailing whitespace from the accumulated text, which commonly appears due to blank lines at the start or end of email bodies.

---

### Step 3: Configure Paths

```python
root_folder = r"C:\Emails_Archiving\[ARCHIVE_FOLDER_NAME]"
output_csv  = r"C:\Emails_Archiving\emails_parsed.csv"
```

These are the only two variables you need to change when running this notebook on a different archive. Both are plain strings — the raw string prefix `r"..."` is used to prevent Python from interpreting the backslashes in Windows paths as escape sequences.

**`root_folder`** should point to the top-level directory that contains your `.eml` files. This can be any folder at any depth; the script will find all `.eml` files underneath it regardless of how deeply nested they are.

**`output_csv`** should point to the desired output location. The notebook will create the parent directory if it does not already exist, so you do not need to create it manually.

A print statement confirms the configured paths before the walk begins, which helps catch typos or path errors early.

---

### Step 4: Walk the Archive and Extract Content

```python
emails_data = []

for dirpath, dirnames, filenames in os.walk(root_folder):
    for filename in filenames:
        if filename.lower().endswith(".eml"):
            file_path = os.path.join(dirpath, filename)
            email_text = extract_email_text(file_path)
            emails_data.append({
                "file_name": filename,
                "file_path": file_path,
                "email_text": email_text
            })

df_emails = pd.DataFrame(emails_data)
```

`os.walk()` is a generator that yields a three-tuple `(dirpath, dirnames, filenames)` for each directory it visits. `dirpath` is the path to the current directory, `dirnames` is a list of subdirectory names in that directory, and `filenames` is a list of non-directory filenames. The generator handles all recursion automatically.

The `.lower().endswith(".eml")` check is case-insensitive — it will match `.eml`, `.EML`, `.Eml`, and any other capitalisation. This is important for robustness since file extensions are not always consistently cased across different operating systems or conversion tools.

Results are accumulated in a Python list (`emails_data`) rather than appended directly to a DataFrame. This is significantly more performant for large numbers of files because pandas DataFrame concatenation in a loop is expensive — each append would require reallocating the entire DataFrame in memory. Building a list and converting once at the end is the standard Python pattern for this kind of data collection.

The three fields stored per email are deliberately minimal: just what is needed to uniquely identify the file (`file_name`, `file_path`) and its content (`email_text`). All further enrichment — headers, folder hierarchy, PII extraction — is deferred to downstream processing.

After the walk, a summary is printed showing total file count, DataFrame shape, column names, and a sample of filenames, providing immediate confirmation that the walk succeeded.

---

### Step 5: Save to CSV

```python
os.makedirs(os.path.dirname(output_csv), exist_ok=True)
df_emails.to_csv(output_csv, index=False)
```

`os.makedirs(..., exist_ok=True)` creates the output directory and any intermediate parent directories if they do not already exist. The `exist_ok=True` argument prevents it from raising an error if the directory already exists.

`to_csv(..., index=False)` writes the DataFrame to CSV without including the pandas integer row index as a column. The downstream pipeline does not use this index (it creates its own indexing after chain splitting and other transformations), and omitting it keeps the output file cleaner and reduces file size slightly.

#### Why a `PermissionError` can occur here

The most common failure at this step is a `PermissionError`. This can happen because:

- **The target directory does not exist** — addressed by the `os.makedirs()` call above
- **The output CSV is open in another application** — Windows locks files that are open in Excel or similar programs; close the file and retry
- **Insufficient write permissions** — on institutional systems, the target directory may be read-only or require elevated permissions; choose a different output location such as your Desktop or Documents folder

---

### Step 6: Spot-Check the Output

After saving, the notebook reloads the CSV from disk to verify it was written correctly. Three validation checks are run:

**Structural check:**
```python
df_check = pd.read_csv(output_csv)
print(df_check.isnull().sum())
print((df_check['email_text'] == '').sum())
```
This confirms the row count matches expectations, no columns have unexpected nulls, and identifies how many emails produced an empty `email_text`. A small number of empty bodies is expected (HTML-only or attachment-only emails), but a large number might indicate a path configuration issue.

**Preview:**
```python
df_check.head()
```
A visual inspection of the first few rows confirms that `file_name`, `file_path`, and `email_text` columns contain sensible values.

**Length distribution check:**
```python
df_check["text_length"] = df_check["email_text"].fillna("").str.len()
print(df_check["text_length"].describe())
print((df_check['text_length'] == 0).sum())
print((df_check['text_length'] < 20).sum())
```
Checking the distribution of body lengths helps identify parsing anomalies. If the mean or median text length is unexpectedly low, it may indicate that most emails are being parsed as empty — which could point to a MIME handling issue (for example, if all emails in the archive are HTML-only with no `text/plain` part).

---

## 7. Understanding EML and MIME

### What is an EML file?

An `.eml` file is a plain-text file containing a single email message in the format defined by [RFC 2822](https://datatracker.ietf.org/doc/html/rfc2822) (and its predecessor RFC 822). It begins with a series of header lines (key-value pairs separated by `: `), followed by a blank line, followed by the message body.

A simple single-part email looks like this:

```
From: sender@example.com
To: recipient@example.com
Date: Mon, 10 Jun 2003 14:44:00 -0500
Subject: Update on project status
Content-Type: text/plain; charset="utf-8"

Hi all,

This is the body of the email. It contains the message content
as plain text.
```

### What is MIME?

MIME (Multipurpose Internet Mail Extensions) is a standard ([RFC 2045–2049](https://datatracker.ietf.org/doc/html/rfc2045)) that extends the basic RFC 2822 format to support:

- Multiple content types in a single message (plain text + HTML + images)
- Non-ASCII character encodings
- Binary file attachments encoded as base64 or quoted-printable text
- Nested message structures

A multipart email has a `Content-Type` header like `multipart/alternative` or `multipart/mixed`, and the body contains multiple "parts" separated by a boundary string. Each part has its own mini-headers and content.

### Why MIME handling matters for this archive

Emails in a 20+ year archive were created by many different clients (early Microsoft Outlook versions, Lotus Notes, web clients, automated systems) under many different configurations. This creates significant MIME diversity:

- Very old emails (late 1990s) are often simple single-part plain text
- Emails from the early 2000s may use `multipart/alternative` with both text and basic HTML
- Some automated system emails may be `multipart/mixed` with text and attachments
- Encoding varies widely: UTF-8, Windows-1252, ISO-8859-1, and sometimes no declared encoding at all

The `extract_email_text()` function handles this diversity by using Python's `email` library (which implements the full MIME specification), reading declared charsets from each part's headers, and falling back gracefully when information is missing.

---

## 8. Design Decisions and Rationale

### Why plain-text only, not HTML?

For archival and text analysis purposes, `text/plain` is the most useful representation. HTML content requires additional parsing to strip tags and decode HTML entities before it can be analyzed. For the downstream PII detection, topic modeling, and LLM summarization steps, clean plain text is far preferable to HTML. In nearly all cases, the plain-text version of an email contains the same information as the HTML version — it is just less visually formatted.

If an email exists only in HTML format (no `text/plain` part), it will produce an empty `email_text`. This is acceptable for the archiving use case — such emails can be identified in the downstream quality checks and handled separately if needed.

### Why collect into a list before building a DataFrame?

Appending to a pandas DataFrame inside a loop (e.g., `df = df.append(row, ...)`) is an extremely common anti-pattern. Each `.append()` call creates a full copy of the entire DataFrame, resulting in O(n²) time and memory complexity as the number of rows grows. For 44,677 emails, this would make the walk dramatically slower.

The correct pattern is to collect records in a Python list, then call `pd.DataFrame(list)` once at the end. The list append is O(1) amortized, and the single DataFrame construction at the end is efficient.

### Why store the full file path rather than just the folder name?

The full path is a richer artifact than just the folder name because it preserves the complete hierarchy. Downstream notebooks can split this path on the OS separator to create `Folder_1`, `Folder_2`, ..., `Folder_N` columns, or use string operations to filter by any level of the hierarchy. If we had stored only the immediate parent folder name, that flexibility would be lost.

### Why `errors='replace'` rather than `errors='ignore'` or `errors='strict'`?

`errors='strict'` (the default) would raise an exception on the first malformed byte, which is unacceptable for a large archive where some emails are expected to have encoding issues.

`errors='ignore'` silently drops malformed bytes, which could result in missing content without any indication that anything was lost.

`errors='replace'` substitutes malformed bytes with the Unicode replacement character (`\ufffd`), which is visible in the output and signals that some content was undecodable. This is the most informative option — it keeps the pipeline running while making encoding issues detectable in the output data.

### Why `index=False` when saving the CSV?

The pandas row index at this stage is just an integer sequence (0, 1, 2, ...) that carries no information. Including it would add a column to the output that downstream code would then need to explicitly drop. By omitting it here, the output CSV is cleaner and immediately usable without cleanup.

---

## 9. What This Notebook Does NOT Capture

This notebook is intentionally narrow. The following are explicitly out of scope and are handled in the downstream notebook:

**Email headers** — From, To, Cc, Subject, Date, Importance, and other header fields are present in the raw `.eml` content but are not extracted here. They are parsed using regular expressions in `Email_Archiving_Data_Preparation_v1_CLEAN.ipynb`.

**HTML email content** — Only `text/plain` MIME parts are extracted. If an email has no plain-text part (HTML-only), `email_text` will be empty for that row.

**File attachments** — Any MIME part with a `Content-Disposition: attachment` header is skipped entirely. Attachment filenames, types, and content are not captured.

**Email metadata beyond file path** — Information like message ID, thread ID, in-reply-to references, and MIME boundary strings are not extracted.

**Email chain splitting** — Many `.eml` files contain entire conversation threads (the reply includes the previous message, which includes the message before that, etc.). The `email_text` column in the output contains the full thread as a single string. Splitting these chains into individual per-message rows is done in the next notebook using the `-----Original Message-----` delimiter.

**PII detection and anonymization** — No personally identifiable information is detected, masked, or removed at this stage.

---

## 10. Known Issues and Edge Cases

**HTML-only emails produce empty `email_text`:** Some email clients (and many automated notification systems) send emails with only a `text/html` MIME part and no `text/plain` equivalent. These will produce an empty string in the `email_text` column. In the full archive, these appear in the spot-check as emails with `text_length == 0`.

**Deeply nested MIME structures:** Some complex emails (meeting invitations, digitally signed messages, encrypted messages) use deeply nested MIME structures. The `msg.walk()` traversal handles arbitrary nesting depth correctly, but the useful text content may be buried inside a `multipart/signed` or `multipart/encrypted` container that wraps a `text/plain` part. In practice, most such emails are still parsed correctly because `walk()` descends into all containers.

**Non-standard delimiters in chained emails:** The string `-----Original Message-----` is the most common delimiter used by Microsoft Outlook when forwarding or replying. However, some email clients use different formats (`> quoted text`, `From: ...` followed by a date line, `________`, etc.). Emails from these clients will not be split correctly in the downstream chain-splitting step, though they will still be captured as a complete block in `email_text`.

**Very large emails:** Emails with large attachments, embedded images, or very long threads may take significantly longer to parse than a typical message. There is no timeout or size limit implemented in the current version.

**Read-only or locked files:** If the archive folder is on a network drive or external storage, individual files may be temporarily unavailable or locked. These will produce `PermissionError` or `IOError` messages from the `except` block in `extract_email_text()`, but the pipeline will continue processing the remaining files. The affected rows will not appear in the output CSV.

---

## 11. Troubleshooting

**`PermissionError` when saving the CSV**

The most common cause is that the output CSV is already open in Excel or another application. Close the file and re-run the save cell. Alternatively, change `output_csv` to point to a different location (e.g., your Desktop).

If the target directory does not exist and the `os.makedirs()` call fails, verify that the parent directory of your intended output path exists and is writable.

**`FileNotFoundError` when setting `root_folder`**

Double-check the path. Common mistakes include extra or missing backslashes, a folder name with a trailing space, or using a relative path instead of an absolute path. Print `os.path.exists(root_folder)` to verify the path is valid before running the walk.

**Walk completes but `len(df_emails)` is 0**

This means no `.eml` files were found under `root_folder`. Check that the folder actually contains `.eml` files (not some other extension), that the path is correct, and that `os.walk()` has permission to read the subdirectories. You can test with a small print loop:

```python
for dirpath, dirnames, filenames in os.walk(root_folder):
    print(dirpath, len(filenames))
    break  # just print the first directory
```

**Many empty `email_text` values**

If a large proportion of emails have empty text content, the most likely cause is that the emails in your archive are HTML-only (no `text/plain` MIME part). You can verify this by examining a sample `.eml` file directly in a text editor and checking the `Content-Type` header. If this is the case, the extraction function would need to be extended to parse `text/html` parts and strip HTML tags — this is outside the current scope of the notebook.

**Slow performance**

For an archive of ~45,000 emails, the walk typically takes several minutes depending on hardware and storage speed. Network drives and older spinning hard disks will be significantly slower than local SSDs. The function prints an error message for any file that fails to parse, so you can monitor progress by watching the output. No progress bar is included in the current version — if you need one, `tqdm` can be added around the `os.walk` loop.

---

## 12. Next Step

The CSV produced by this notebook (`emails_parsed.csv`) is the input to the downstream processing notebook:

**`Email_Archiving_Data_Preparation_v1_CLEAN.ipynb`**

That notebook picks up where this one leaves off, performing:

- **Folder hierarchy extraction** — splitting `file_path` on backslashes to create queryable `Folder_1` through `Folder_N` columns
- **Email title extraction** — parsing the filename to create a clean `email_title` column
- **Email chain splitting** — exploding multi-message threads into individual rows (expanding ~44,677 files to ~79,676 message segments)
- **Header parsing** — extracting From, Sent, To, Cc, Subject, and Importance fields using regular expressions
- **Fuzzy sender search** — filtering emails by sender name with `fuzzywuzzy` partial ratio matching
- **Temporal analysis** — analyzing email volume over time using parsed datetime fields
- **Topic modeling** — discovering content themes via LDA and labeling them with Llama 3.2
- **PII detection and anonymization** — using Microsoft's `presidio-analyzer` to detect and replace names, phone numbers, email addresses, dates, and SSNs
- **LLM-based summarization** — rewriting each email as an anonymized summary using Llama 3.2 via Ollama

See that notebook and its accompanying README for full documentation of those steps.

---

*University of Illinois System | Records and Information Management Services | Decision Support*
