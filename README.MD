# Student Data Cleaning & Filtering Workflow (n8n)

An n8n automation that takes a messy student CSV, processes it one record at a time, cleans it, filters it to valid Chennai students, and exports the result as a CSV file.

## Workflow Overview

```
CSV upload → Extract from File → Basic LLM Chain (OpenAI) → Loop Over Items
                                                                 │
                                          ┌──────────────────────┴───────────┐
                                        loop                               done
                                          ↓                                  ↓
                                  Code in JavaScript                        If
                                  (cleaning, per record)             ┌───────┴───────┐
                                          │                        true            false
                                          └→ back to loop            ↓          (rejected)
                                                              Convert to File
                                                              (CSV download)
```

## What Each Node Does

| Node | Purpose |
|---|---|
| **CSV upload** | Entry point; receives the student CSV file |
| **Extract from File** | Converts CSV rows into JSON items (50 records) |
| **Basic LLM Chain + OpenAI Chat Model** | AI-assisted processing of the records |
| **Loop Over Items** | Sends records one by one (batch size 1) |
| **Code in JavaScript** | Cleans each record (trim, lowercase email, handle `MISSING` values) |
| **If** | Keeps only records where `City = Chennai` **AND** `Email` contains `@` |
| **Convert to File** | Exports the valid records as `cleaned_students.csv` |

## Input Data Fields

`Student_ID`, `Name`, `Email`, `Phone`, `Course`, `Fee_Paid`, `City`, `Enrolled_Date`

The data intentionally contains issues such as `INVALID_EMAIL` and `MISSING` phone numbers.

## Filter Logic

| Condition | Operator | Value |
|---|---|---|
| `{{ $json.City }}` | is equal to | `Chennai` |
| `{{ $json.Email }}` | **contains** | `@` |

Optional stricter email check (regex): `^[^\s@]+@[^\s@]+\.[^\s@]+$`

## How to Use

1. Import the workflow JSON into n8n (**Import from File**).
2. Add your own OpenAI credentials to the **OpenAI Chat Model** node.
3. Upload your CSV in the **CSV upload** node.
4. Click **Execute workflow**.
5. Open **Convert to File** → **Binary** tab → **Download**.

## Key Learnings / Troubleshooting

- **"is not equal to @" is not an email check.** It only rejects the exact text `@`, so `INVALID_EMAIL` passed. Use **contains** (or regex) instead.
- **Pinned data causes duplicates.** A pinned Code node returned all 50 records on every loop pass, giving 50 × 50 = 2500 items. Unpin nodes before real runs.
- **Close the loop.** The last node in the loop must connect back to Loop Over Items, otherwise only the first record is processed.
- **Loop vs. Done outputs.** The *loop* output handles each record; the *done* output runs once after all records finish.
- **File downloads as a generic "file".** Set **Options → File Name** in Convert to File with the extension (e.g. `cleaned_students.csv`).
- **n8n runs nodes per item by default.** A loop is only needed for delays, per-record error handling, or APIs that accept one record at a time.

## Tech Stack

n8n, JavaScript, OpenAI API

