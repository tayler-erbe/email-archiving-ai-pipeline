# Sample Email Archive Dataset (Dummy Data)

## Overview
This dataset is a **dummy/sample version** of the original email archiving dataset. It is designed to **mimic the structure, column naming conventions, and data types** of the real data while containing no sensitive or real information.

The purpose of this dataset is to:
- demonstrate the schema of the original data  
- provide a safe example for development and testing  
- illustrate how the data is structured for downstream processing  

This dataset does **not** contain real email content or personally identifiable information.

---

## Data Structure

Below is a description of the columns included in this dataset.

| Column Name | Description | Data Type |
|------------|------------|----------|
| `email_id` | Unique identifier for each email record | String |
| `sender` | Email address of the sender | String |
| `recipient` | Email address of the primary recipient | String |
| `cc` | Carbon copy recipients (if applicable) | String |
| `bcc` | Blind carbon copy recipients (if applicable) | String |
| `subject` | Email subject line | String |
| `body` | Parsed email body content | String |
| `timestamp` | Date and time the email was sent | Datetime |
| `file_path` | Original file location or reference path | String |
| `has_attachments` | Indicator for presence of attachments | Boolean |
| `attachment_count` | Number of attachments associated with the email | Integer |
| `pii_detected` | Flag indicating whether PII was detected | Boolean |
| `pii_entities` | Extracted PII elements (if any) | String / List |
| `cleaned_body` | Processed and normalized version of email body | String |
| `redacted_body` | Email body after PII redaction | String |
| `summary` | Generated summary of the email content | String |

---

## Notes

- All values in this dataset are **synthetic or anonymized**.
- The schema reflects the structure used in the production pipeline.
- This dataset is intended for **demonstration, development, and testing purposes only**.

---

## Usage

You can use this dataset to:
- test data pipelines  
- validate schema expectations  
- prototype NLP or ML workflows  
- demonstrate functionality without exposing sensitive data  

---

## Author
Tayler Erbe  
Senior Data Scientist