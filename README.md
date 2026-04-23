# Authorization Document Reader

An AI-powered data extraction pipeline that automatically ingests 
authorization PDFs from school districts and writes structured 
data into a Google Sheet masterfile — eliminating manual data 
entry entirely.

Built with n8n, Claude AI, OpenRouter, and Google Workspace.

---

## The Problem

School districts send authorization documents in completely 
different formats. Field names vary, layouts differ, and some 
PDFs are clean text while others are scanned images. Manual 
data entry is slow, error-prone, and doesn't scale.

---

## The Solution

A fully automated pipeline that:
- Detects whether a PDF is text-based or scanned automatically
- Routes each PDF to the appropriate extraction path
- Uses Claude AI to extract structured data regardless of format
- Validates every field and logs warnings without crashing
- Writes directly to the correct row and column in Google Sheets

---

## How It Works
Google Drive Trigger (new PDF uploaded)
↓
Download PDF
↓
Extract Text from PDF
↓
Clean Text (strip whitespace and newlines)
↓
IF — Real content found?
├── YES → Claude Haiku via OpenRouter (text extraction)
└── NO  → Redownload PDF
↓
Claude Haiku via Anthropic Vision (scanned extraction)
↓
Parse and Validate (3-layer error handling)
↓
Read all rows from Google Sheet
↓
Find student by name (case-insensitive)
↓
IF — Student found?
├── YES → Build Update Payload
│             ↓
│         IF — Auth slot available?
│         ├── YES → Update existing row
│         └── NO  → Log warning
└── NO  → Append new row
↓
IF — Has warnings?
└── YES → Log to Warnings sheet

---

## Fields Extracted

| Field | Description |
|---|---|
| student_name | Full name of student or client |
| student_id | Client ID or UCI number |
| guardian_name | Name of parent or guardian |
| authorization_number | Authorization number |
| district | School district or agency name |
| service_type | Type of service authorized |
| authorized_hours_per_month | Number of hours per month |
| start_date | Service start date (MM/DD/YY) |
| end_date | Service end date (MM/DD/YY) |
| case_manager_name | Caseworker or case manager name |
| subject_areas | Subject areas or service description |
| gross_auth_amount | Total authorized dollar amount |
| notes | Additional notes or comments |

---

## Tech Stack

| Tool | Role |
|---|---|
| n8n | Workflow orchestration |
| Google Drive | PDF storage and trigger |
| Google Sheets | Masterfile output and warnings log |
| Claude Haiku via OpenRouter | Text PDF extraction |
| Claude Haiku via Anthropic API | Scanned PDF vision extraction |

---

## Key Design Decisions

**Dual extraction path**
Text PDFs are cheaper and faster to process via text extraction. 
Vision is only invoked when genuinely needed — keeping costs low 
while maintaining reliability for all document types.

**Whitespace cleaning before routing**
Scanned PDFs often produce extraction output containing only 
newline characters. Stripping whitespace before checking content 
length ensures accurate routing regardless of how the PDF 
renderer handles whitespace.

**Redownload for vision path**
Binary data doesn't survive cleanly through n8n routing nodes. 
Redownloading the file fresh on the vision path eliminates binary 
handling errors without complex workarounds.

**Smart authorization column detection**
The sheet tracks up to four authorization periods per student. 
The system checks each column in order and writes to the first 
available one — preserving existing records while adding the new 
entry in the correct position.

**Append comments, never overwrite**
Authorization comments contain cumulative history spanning 
multiple periods. The system appends new comments to existing 
ones rather than replacing them — preserving the full audit trail.

---

## Edge Cases Handled

- ✅ Whitespace-only PDF extraction routed correctly to vision
- ✅ Cumulative authorization comments preserved not overwritten
- ✅ Smart detection of which authorization date column to populate
- ✅ Case manager name reformatted from Last, First to First Last
- ✅ New students appended automatically if not found in sheet
- ✅ All four authorization columns full — warning logged for review
- ✅ Missing fields logged without crashing
- ✅ Unparseable LLM responses caught and logged gracefully

---

## Setup Instructions

### Prerequisites
- n8n instance (self-hosted or cloud)
- Google account with Drive and Sheets access
- OpenRouter account and API key
- Anthropic account and API key

### Step 1 — Google Drive Setup
1. Create a folder in Google Drive called **Auth PDFs**
2. Note the folder ID from the URL

### Step 2 — Google Sheets Setup
1. Make a copy of the masterfile template
2. Add a second sheet tab called **Warnings** with columns:
Timestamp | Student Name | Auth Number | File Name | Warnings

### Step 3 — n8n Credentials
Set up these credentials in n8n:
- Google Drive OAuth2
- Google Sheets OAuth2
- OpenRouter API Key
- Anthropic API Key

### Step 4 — Import Workflow
1. Open n8n
2. Go to **Workflows** → **Import**
3. Upload `workflow/TutorMe_Authorization_Document_Reader.json`
4. Update the following in the imported workflow:
   - **Auth Letter detector** — select your Auth PDFs folder
   - **Get row(s) in sheet** — select your Google Sheet
   - **Update existing row** — select your Google Sheet
   - **Append new row** — select your Google Sheet
   - **Append to Warnings** — select your Warnings sheet
   - **Anthropic Vision** — add your Anthropic API key to headers
   - **OpenRouter Chat Model** — select your OpenRouter credential

### Step 5 — Test
1. Upload the sample PDF to your Auth PDFs folder
2. Watch the workflow trigger automatically
3. Verify the Google Sheet is updated correctly
4. Check the Warnings sheet for any flagged issues

---

## Warnings Sheet

Any document with issues is logged here without stopping the 
pipeline:

| Column | Description |
|---|---|
| Timestamp | Date the document was processed |
| Student Name | Extracted student name |
| Auth Number | Extracted authorization number |
| File Name | Original PDF filename |
| Warnings | Specific warning messages |

---

## What I'd Improve With More Time

- **Duplicate detection** — cross-check auth number against 
  existing comments before writing to prevent duplicate entries
- **Email trigger** — monitor Gmail inbox directly and process 
  PDF attachments without manual upload
- **Confidence scoring** — second Claude call to rate confidence 
  per extracted field and flag low-confidence values for review
- **Multi-student documents** — detect and handle PDFs containing 
  multiple students in a single document

---

## Author

**Eljon G. Mateo**
AI Automation Specialist
mateoeljon@gmail.com
eljonmateo.dev
