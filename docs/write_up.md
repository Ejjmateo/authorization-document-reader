# Authorization Document Reader — Technical Write Up
**Eljon G. Mateo | AI Automation Specialist**
mateoeljon@gmail.com | eljonmateo.dev

---

## What I Built

A fully automated data extraction pipeline that monitors a Gmail
inbox for incoming authorization PDFs from school districts and
writes structured data directly into a Google Sheet masterfile
— eliminating manual data entry entirely.

The system works in two paths. When a new email with a PDF
attachment arrives the Gmail trigger fires automatically. A native
Gmail node downloads the attachment binary and passes it to the
PDF text extraction node. If meaningful text is found the content
is sent to Claude Haiku via OpenRouter for fast structured
extraction. If the extracted text contains only whitespace —
indicating a scanned document — a second Gmail node re-fetches
the attachment binary fresh, converts it to base64, and sends
it to the Anthropic Claude API using the dedicated PDF document
type for vision-based extraction. Both paths feed into the same
validation and Google Sheet update logic.

Before writing to the sheet the system validates every field,
detects which authorization date column to populate based on
what is already filled, appends new comments to existing ones
rather than overwriting, formats case manager names from
Last, First to First Last, and logs any missing fields or issues
to a dedicated Warnings sheet — without crashing or blocking
subsequent documents.

---

## 2–3 Tradeoffs I Considered

**1. Re-fetching the attachment vs passing binary through the IF node**
My initial approach tried to pass the Gmail attachment binary
through the IF routing node to the vision path. The problem was
that n8n does not reliably carry binary data through routing nodes
— the binary reference gets lost and the vision node receives
nothing. I considered using a Merge node to combine the routing
decision with the binary data, but the cleanest and most reliable
solution was using a dedicated Gmail Download Attachment node on
each path independently. This adds one extra Gmail API call for
scanned documents but completely eliminates the binary handling
errors — and makes each path self-contained and easy to debug.

**2. Single extraction path vs dual path**
Using Claude's vision API for every document regardless of type
would have simplified the workflow significantly — one path, no
routing logic needed. I chose the dual path approach because text
extraction is meaningfully faster and cheaper for clean PDFs,
and the vision API should only be invoked when genuinely needed.
For a system processing hundreds of documents over time the cost
difference is significant. The tradeoff is added workflow
complexity and the whitespace cleaning logic needed to correctly
detect scanned documents.

**3. Google Sheets Lookup node vs reading all rows**
n8n's native Sheets Lookup node has limitations with partial
matches and case sensitivity. I chose to read all rows and perform
the student matching in a Code node using case-insensitive
comparison instead. This gives full control over matching logic
and handles edge cases like inconsistent capitalization in student
names. The tradeoff is that for very large sheets this reads
every row on every trigger — but for the current dataset size
this is not a performance concern.

---

## What I'd Improve With Another 10 Hours

**1. Attachment loop for multiple PDFs**
The current system processes the first PDF attachment in an email.
Some districts may send multiple authorizations in a single email.
With more time I'd add a loop that iterates through all PDF
attachments and processes each one independently — so a single
email with five authorizations produces five separate sheet
updates without any manual intervention.

**2. Duplicate authorization detection**
The current system always writes a new authorization comment and
date when a document is processed. With more time I'd extract
the authorization number and cross-check it against existing
authorization comments before writing — detecting whether this
is a truly new authorization or a re-send of one already
processed, and skipping duplicates automatically.

**3. Confidence scoring on extracted fields**
The current system flags fields as missing when they return null
but doesn't flag fields where the LLM extracted a value it may
not be confident about. I'd add a second Claude call that reviews
the extracted JSON against the original document and rates
confidence per field — flagging low-confidence extractions for
human review even when a value was found.

---

## Edge Cases I Noticed in the Data

**1. Whitespace-only PDF extraction**
The sample scanned PDF produced extraction output containing only
newline characters — which technically exists as a string but
carries no real content. A naive text exists check would have
incorrectly routed this to the text extraction path and sent
empty content to Claude. I added a whitespace stripping step
before the length check to correctly detect and route these
documents to the vision path.

**2. Authorization comments are cumulative not replaceable**
Looking at existing sheet data authorization comments span
multiple authorization periods and contain detailed history.
Overwriting this field would destroy critical records. I changed
the logic to append new comments to existing ones rather than
replace them — preserving the full audit trail across all
authorization periods.

**3. Multiple authorization periods per student**
The sheet has four separate authorization date columns — 1st
through 4th Auth. A system that blindly writes to the first
column would overwrite existing records. I built logic to detect
which columns are already populated and write to the next
available one — with a warning logged if all four are full.

**4. Case manager name format inconsistency**
The sample PDF listed the caseworker as SMITH, JORDAN in Last,
First format while the sheet stores names as First Last. I added
formatting logic to detect and convert the comma-separated format
automatically — ensuring consistency with existing records.

**5. Student not found in masterfile**
The system handles the case where an authorization arrives for a
student who does not yet exist in the sheet by appending a new
row rather than failing silently — ensuring no authorization is
ever dropped regardless of whether the student record exists.

**6. Binary data lost through routing nodes**
During testing the PDF binary data was not reliably available
on the vision fallback path after passing through the IF routing
node. Rather than complex workarounds a dedicated Gmail Download
Attachment node re-fetches the binary independently on the vision
path — ensuring it is always fresh and properly available for
base64 conversion.

---

*Built with n8n • Claude Haiku • OpenRouter • Anthropic API • Gmail API • Google Sheets API*
*Eljon G. Mateo — AI Automation Specialist | eljonmateo.dev*
