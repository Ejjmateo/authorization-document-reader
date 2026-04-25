Authorization Document Reader
An AI-powered data extraction pipeline that automatically ingests
authorization PDFs from school district emails and writes structured
data into a Google Sheet masterfile — eliminating manual data entry
entirely.
Built with n8n, Claude AI, OpenRouter, and Google Workspace.

The Problem
School districts send authorization documents in completely
different formats. Field names vary, layouts differ, and some
PDFs are clean text while others are scanned images. Manual
data entry is slow, error-prone, and doesn't scale.

The Solution
A fully automated pipeline that:

Triggers automatically when an email with a PDF attachment arrives
Detects whether a PDF is text-based or scanned automatically
Routes each PDF to the appropriate extraction path
Uses Claude AI to extract structured data regardless of format
Validates every field and logs warnings without crashing
Writes directly to the correct row and column in Google Sheets


How It Works
Gmail Trigger (new email with PDF attachment received)
      ↓
Gmail — Download Attachment (downloads PDF binary)
      ↓
Prepare Payload (pass binary to extract node)
      ↓
Extract Text from PDF
      ↓
Clean Text (strip whitespace and newlines)
      ↓
IF — Real content found?
├── YES → Claude Haiku via OpenRouter (text extraction)
└── NO  → Gmail — Download Attachment for Claude Vision
              ↓
          Prepare Payload for Vision (pass binary)
              ↓
          Prepare PDF for Vision (convert to base64)
              ↓
          Claude Haiku via Anthropic Vision API
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

How to Trigger the Workflow
Send an email to your connected Gmail address with the
authorization PDF attached. The subject line can be anything
— the trigger filters by attachment type not subject line.
Example:

To: yourgmail@gmail.com
Subject: Auth Letter - Alex Rivera
Attachment: authorization.pdf

The workflow triggers automatically within 1 minute of the
email arriving.
Supported attachment formats:

Any file ending in .pdf
Any file with application/pdf mime type


Fields Extracted
FieldDescriptionstudent_nameFull name of student or clientstudent_idClient ID or UCI numberguardian_nameName of parent or guardianauthorization_numberAuthorization numberdistrictSchool district or agency nameservice_typeType of service authorizedauthorized_hours_per_monthNumber of hours per monthstart_dateService start date (MM/DD/YY)end_dateService end date (MM/DD/YY)case_manager_nameCaseworker or case manager namesubject_areasSubject areas or service descriptiongross_auth_amountTotal authorized dollar amountnotesAdditional notes or comments

Tech Stack
ToolRolen8nWorkflow orchestrationGmailEmail trigger and PDF sourceGoogle SheetsMasterfile output and warnings logClaude Haiku via OpenRouterText PDF extractionClaude Haiku via Anthropic APIVision extraction for scanned PDFs

Node Structure
NodeTypePurposeGmail - New Auth EmailGmail TriggerDetects new emails with PDF attachmentsDownload AttachmentGmailDownloads PDF binary from emailPrepare PayloadSetPasses binary to extract nodeExtract Text from PDFExtract from FileExtracts raw text from PDFTrim the whitespace and newlinesCodeCleans text and detects if content existsIFIFRoutes text vs scanned PDFsClaude ParserBasic LLM ChainExtracts fields from text PDFsOpenRouter Chat ModelOpenRouterClaude Haiku model for text extractionDownload Attachment for Claude VisionGmailRe-fetches PDF binary for vision pathPrepare Payload for VisionSetPasses binary to vision nodePrepare PDF for VisionCodeConverts binary to base64Anthropic VisionHTTP RequestSends PDF to Claude vision APIParse and Validate ResultCodeValidates fields and formats outputGet row(s) in sheetGoogle SheetsReads all rows from masterfileCheck if the student existsCodeFinds student by nameCheck if Old or New StudentIFRoutes to update or appendBuild Google Sheet PayloadCodeFormats data for sheet updateIF Auth Slot Available?IFChecks which auth column to write toWith Auth SlotGoogle SheetsUpdates existing rowWith No Auth SlotGoogle SheetsUpdates row with overflow warningPayload for New StudentCodeFormats data for new rowAdd New StudentGoogle SheetsAppends new student rowCheck for Warnings (x3)IFChecks if warnings existLog Warnings (x3)Google SheetsLogs warnings to Warnings sheet

Key Design Decisions
Dual extraction path
Text PDFs are cheaper and faster to process via text extraction.
Vision is only invoked when genuinely needed — keeping costs low
while maintaining reliability for all document types.
Whitespace cleaning before routing
Scanned PDFs often produce extraction output containing only
newline characters. Stripping whitespace before checking content
length ensures accurate routing regardless of how the PDF
renderer handles whitespace.
Native Gmail nodes for attachment handling
Instead of routing binary data through IF nodes — which causes
n8n to lose the binary reference — the workflow uses dedicated
Gmail Download Attachment nodes on each path. This ensures the
PDF binary is always fresh and properly available regardless of
which extraction path is taken.
Smart authorization column detection
The sheet tracks up to four authorization periods per student.
The system checks each column in order and writes to the first
available one — preserving existing records while adding the
new entry in the correct position.
Append comments, never overwrite
Authorization comments contain cumulative history spanning
multiple periods. The system appends new comments to existing
ones rather than replacing them — preserving the full audit trail.

Edge Cases Handled

✅ Whitespace-only PDF extraction routed correctly to vision
✅ Cumulative authorization comments preserved not overwritten
✅ Smart detection of which authorization date column to populate
✅ Case manager name reformatted from Last, First to First Last
✅ New students appended automatically if not found in sheet
✅ All four authorization columns full — warning logged for review
✅ Missing fields logged without crashing
✅ Unparseable LLM responses caught and logged gracefully


Setup Instructions
Prerequisites

n8n instance (self-hosted or cloud)
Gmail account to use as the authorization inbox
Google Sheets access
OpenRouter account and API key
Anthropic account and API key

Step 1 — Gmail Setup

Decide which Gmail account will receive authorization PDFs
Share that email address with whoever sends authorization documents
Make sure the account is accessible via Google OAuth2

Step 2 — Google Sheets Setup

Make a copy of the masterfile template
Add a second sheet tab called Warnings with these columns:

Timestamp | Student Name | Auth Number | File Name | Warnings
Step 3 — n8n Credentials
Set up these credentials in n8n:

Gmail OAuth2
Google Sheets OAuth2
OpenRouter API Key
Anthropic API Key

Step 4 — Import Workflow

Open n8n
Go to Workflows → Import
Upload workflow/TutorMe_Authorization_Document_Reader_v2.json
Update the following nodes with your credentials:

Gmail - New Auth Email — connect Gmail OAuth2
Download Attachment — connect Gmail OAuth2
Download Attachment for Claude Vision — connect Gmail OAuth2
Get row(s) in sheet — connect Google Sheets OAuth2 and select your sheet
With Auth Slot — connect Google Sheets OAuth2 and select your sheet
With No Auth Slot — connect Google Sheets OAuth2 and select your sheet
Add New Student — connect Google Sheets OAuth2 and select your sheet
Log Warnings (x3) — connect Google Sheets OAuth2 and select Warnings sheet
Anthropic Vision — add your Anthropic API key to headers
OpenRouter Chat Model — connect OpenRouter credential



Step 5 — Test

Send an email with the sample Alex Rivera PDF attached to your connected Gmail
Watch the workflow trigger automatically within 1 minute
Verify the Google Sheet is updated correctly
Check the Warnings sheet for any flagged issues


Warnings Sheet
Any document with issues is logged here without stopping the
pipeline:
ColumnDescriptionTimestampDate the document was processedStudent NameExtracted student nameAuth NumberExtracted authorization numberFile NameOriginal PDF filenameWarningsSpecific warning messages

What I'd Improve With More Time

Attachment loop — the current system processes the first
PDF in an email. A future version could loop through all
attachments and process each one independently — useful for
districts that send multiple authorizations in one email
Duplicate detection — cross-check auth number against
existing comments before writing to prevent duplicate entries
Confidence scoring — second Claude call to rate confidence
per extracted field and flag low-confidence values for review
Multi-student documents — detect and handle PDFs containing
multiple students in a single document


Author
Eljon G. Mateo
AI Automation Specialist
mateoeljon@gmail.com
eljonmateo.dev
