Screen and score resumes from Gmail with PDF parsing, HubSpot, Slack and PostgreSQL

https://n8nworkflows.xyz/workflows/screen-and-score-resumes-from-gmail-with-pdf-parsing--hubspot--slack-and-postgresql-13034


# Screen and score resumes from Gmail with PDF parsing, HubSpot, Slack and PostgreSQL

## 1. Workflow Overview

**Title:** Screen and score resumes from Gmail with PDF parsing, HubSpot, Slack and PostgreSQL

**Purpose:**  
This workflow monitors a Gmail inbox for incoming resume submissions, validates and parses PDF attachments into text, extracts structured candidate information with a deterministic “AI-style” rules/parser, computes a qualification score and tier, then routes candidates through acceptance/rejection automation. All candidates and outcomes are logged to PostgreSQL, and operational notifications are sent to Slack. Qualified resumes are archived to Google Drive and pushed to HubSpot.

**Target use cases**
- Automated screening for high-volume resume intake (email-based applications)
- Standardized candidate scoring and tiering (A+…D) using consistent rules
- Centralized logging for compliance/analytics + operational alerts to recruiters

### 1.1 Phase 1 — Intake & Validation
Monitors Gmail, checks attachments, selects a PDF resume, validates file size, and prepares metadata for downstream parsing.

### 1.2 Phase 2 — PDF Parsing & Structured Extraction
Converts the PDF into text and extracts email/phone/LinkedIn/location, skills, experience, education, certifications, and keyword matches. Computes score and tier.

### 1.3 Phase 3 — Smart Routing & Integrations
Branches on qualification threshold (>= 70). Qualified candidates are sent to HubSpot, archived to Drive, and announced on Slack. Rejected candidates get an automated rejection email, archived to Drive, and logged to Slack.

### 1.4 Phase 4 — Analytics & Storage
Both branches converge into a shared analytics computation and a PostgreSQL insert for funnel analytics.

---

## 2. Block-by-Block Analysis

### Block A — Intake & Validation (Phase 1)

**Overview:**  
Receives new Gmail messages, validates that a PDF resume exists and is within size constraints, and enriches the event with standardized fields used downstream.

**Nodes involved:**
- Gmail Trigger
- Code: Pre-Validation
- IF: Valid PDF Attachment?

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — entry point; polls Gmail for new messages.
- **Key configuration:**
  - Poll schedule: **everyMinute**
  - No explicit filters configured (will poll the mailbox broadly).
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** to **Code: Pre-Validation**
- **Credentials:** Gmail OAuth2 (named `jitesh0dugar@gmail.com`)
- **Edge cases / failures:**
  - Gmail OAuth scope/consent issues, token expiry
  - High-volume polling may hit Gmail API usage limits
  - Without filters, non-resume emails may enter the pipeline

#### Node: Code: Pre-Validation
- **Type / role:** `n8n-nodes-base.code` — normalizes input, selects a PDF attachment, validates size, and sets processing flags.
- **Key configuration choices (interpreted):**
  - Iterates over incoming items (`$input.all()`).
  - If **no attachments** → sets:
    - `validationError: "No attachments found"`
    - `skipProcessing: true`
  - Searches for PDF by:
    - `mimeType === 'application/pdf'` OR filename ends with `.pdf`
  - If **no PDF found** → sets:
    - `validationError: "No PDF resume found"`
    - `skipProcessing: true`
  - Validates file size:
    - Reject if > **10MB** (10240 KB)
    - Adds `fileSizeKB` string
  - Extracts/standardizes metadata:
    - `applicantEmail` from `item.json.from.address` (fallback `user@example.com`)
    - `applicantName` from `item.json.from.name` (fallback `Unknown Applicant`)
    - `receivedDate` from `item.json.date` (fallback now)
    - `emailSubject` from `item.json.subject`
    - `processingTimestamp`
  - Stores selected attachment as `pdfAttachment`
- **Expressions / variables used:** standard Code node variables (`$input`, `item.json`, `item.binary`)
- **Inputs / outputs:**
  - **Input:** from **Gmail Trigger**
  - **Output:** to **IF: Valid PDF Attachment?**
- **Edge cases / failures:**
  - Gmail trigger attachment structure may differ; expects `item.json.attachments[]` with `mimeType`, `fileName`, `size`
  - Sets `skipProcessing` but the next IF node does not check it (so “skip” is informational unless used later)
  - If multiple PDFs exist, it chooses the **first matching** PDF only

#### Node: IF: Valid PDF Attachment?
- **Type / role:** `n8n-nodes-base.if` — checks that attachments exist and that the **first** attachment is a PDF.
- **Key configuration choices:**
  - Condition 1: `$json.attachments` **not empty**
  - Condition 2: `$json.attachments[0].mimeType === 'application/pdf'`
- **Inputs / outputs:**
  - **Input:** from **Code: Pre-Validation**
  - **True output:** to **PDF to Text: Extract Content**
  - **False output:** not connected (workflow effectively stops for invalid cases)
- **Edge cases / failures:**
  - Logic mismatch with Pre-Validation:
    - Pre-Validation finds a PDF anywhere in the attachments, but this IF checks only `attachments[0]`.
    - If the PDF is not first, valid resumes may be dropped.
  - No explicit path for invalid items (no rejection email/log for “no PDF” or “oversize”)

---

### Block B — PDF Parsing & Candidate Scoring (Phase 2)

**Overview:**  
Parses the PDF into text and runs deterministic extraction/scoring logic to produce structured candidate fields plus a qualification score and tier.

**Nodes involved:**
- PDF to Text: Extract Content
- Code: AI Resume Parser
- PostgreSQL: Log Application

#### Node: PDF to Text: Extract Content
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — external PDF parsing service (“Parse PDF to JSON”).
- **Key configuration choices:**
  - Resource: `pdfManipulation`
  - Operation: `parsePdfToJson`
  - Expected to output extracted text under `item.json.text` (as used by downstream parser)
- **Inputs / outputs:**
  - **Input:** from **IF: Valid PDF Attachment?** (true branch)
  - **Output:** to **Code: AI Resume Parser**
- **Credentials:** `htmlcsstopdfApi` (named `pdf munk - deepanshi`)
- **Edge cases / failures:**
  - Service authentication issues / quota limits
  - Large or scanned/image PDFs may yield poor text extraction
  - Node must receive the PDF binary/attachment in the format expected by the htmlcsstopdf integration (not explicitly mapped here)

#### Node: Code: AI Resume Parser
- **Type / role:** `n8n-nodes-base.code` — rule-based NLP extraction and scoring.
- **Key configuration choices (interpreted):**
  - Reads `const text = item.json.text || ''`
  - **Contact extraction**
    - Email via regex; fallback to `applicantEmail`
    - Phone via multiple regex patterns; default `Not Found`
    - LinkedIn via `linkedin.com/in/...`; default `Not Found`
    - Location via `City, ST` pattern; default `Not Found`
  - **Skills extraction**
    - Categorized lists (programming/frontend/backend/database/cloud/tools/data)
    - For each skill: regex `\bSKILL\b` (note: some skills include punctuation like `C++`, `.NET`, `Node.js` which can reduce matching accuracy with `\b`)
  - **Experience extraction**
    - Year ranges like `2018-2022` or `2020-Present` summed into `totalYears`
    - Fallback: `(\d+)+ years experience`
  - **Education extraction**
    - Degree keyword patterns => `highestDegree` (PhD/Masters/Bachelors/Associate/None)
    - University matching is simplistic and may miss many formats
  - **Certifications extraction**
    - Fixed list (AWS Certified, PMP, etc.), max bonus capped
  - **Scoring (0–100)**
    - Skills: `min(totalSkills * 4, 40)`
    - Experience: up to 30
    - Education: up to 20
    - Certifications: `min(count * 5, 10)`
    - Qualified if score >= 70
    - Tier mapping: A+/A/B/C/D
  - **Enrichment output fields written into `item.json`:**
    - `candidateEmail`, `candidatePhone`, `candidateLinkedIn`, `candidateLocation`
    - `skills`, `skillsByCategory`, `totalSkills`
    - `yearsExperience`, `highestDegree`, `hasDegree`, `university`
    - `certifications`, `matchedKeywords`
    - `qualificationScore`, `qualified`, `tier`, `tierDescription`
    - `candidateName` from `applicantName`
    - `analysisTimestamp`
- **Inputs / outputs:**
  - **Input:** from **PDF to Text: Extract Content**
  - **Outputs:** to
    - **PostgreSQL: Log Application**
    - **IF: Qualified Candidate?**
- **Edge cases / failures:**
  - If `item.json.text` is missing/empty → extraction yields mostly defaults; still produces a score
  - Regex limitations for skills with special characters (`C++`, `.NET`, `Node.js`)
  - Summing overlapping year ranges can overcount experience if multiple ranges overlap
  - Location regex is US-centric (City, ST)

#### Node: PostgreSQL: Log Application
- **Type / role:** `n8n-nodes-base.postgres` — inserts candidate application record, returns inserted id.
- **Key configuration choices:**
  - Operation: Execute SQL query
  - Inserts into `candidate_applications` with fields:
    - email, name, phone, linkedin, location, skills(JSON string), years_experience, highest_degree, university, certifications(JSON string), qualification_score, tier, qualified, received_date, processed_date(NOW())
  - Returns `id`
- **Inputs / outputs:**
  - **Input:** from **Code: AI Resume Parser**
  - **Output:** none connected (logging side-effect only)
- **Credentials:** Postgres credentials required (not shown in JSON snippet but implied by node)
- **Edge cases / failures:**
  - **SQL injection / quoting risk:** values are interpolated directly into SQL strings (e.g., candidate name containing `'` will break the query). This is a major reliability issue.
  - JSON strings may contain quotes/newlines; may also break SQL unless properly escaped
  - Table/schema must exist; type mismatch (boolean, numeric) can fail inserts
  - Connection pool / network timeouts

---

### Block C — Routing & Integrations (Phase 3)

**Overview:**  
Branches based on `qualified` boolean. Qualified candidates get HubSpot + Drive archive + Slack alert. Rejected candidates get Drive archive + rejection email + Slack log.

**Nodes involved:**
- IF: Qualified Candidate?
- HubSpot: Create Contact
- Google Drive: Archive Qualified
- Slack: Qualified Alert
- Google Drive: Archive Rejected
- Gmail: Send Rejection
- Slack: Rejection Log
- Merge: Qualified Path
- Merge: Rejected Path

#### Node: IF: Qualified Candidate?
- **Type / role:** `n8n-nodes-base.if` — decision gate on candidate qualification.
- **Key configuration choices:**
  - Condition: `$json.qualified === true`
- **Inputs / outputs:**
  - **Input:** from **Code: AI Resume Parser**
  - **True branch outputs:** to
    - HubSpot: Create Contact
    - Google Drive: Archive Qualified
  - **False branch outputs:** to
    - Google Drive: Archive Rejected
    - Gmail: Send Rejection
- **Edge cases / failures:**
  - If `qualified` is missing (e.g., parser failed) it will go false branch by default behavior

#### Node: HubSpot: Create Contact
- **Type / role:** `n8n-nodes-base.hubspot` — creates a contact in HubSpot.
- **Key configuration choices:**
  - Operation: Create
  - Authentication: App Token
  - (No properties mapping is shown; with minimal config this node may create an empty contact unless additional fields are set in the node UI. The JSON provided does not show contact properties mapping.)
- **Inputs / outputs:**
  - **Input:** from **IF: Qualified Candidate?** (true)
  - **Output:** to **Slack: Qualified Alert**
- **Credentials:** HubSpot App Token
- **Edge cases / failures:**
  - Missing required fields (HubSpot often requires at least email)
  - Duplicate contact handling (may error or update depending on HubSpot API/node settings)
  - Rate limits

#### Node: Google Drive: Archive Qualified
- **Type / role:** `n8n-nodes-base.googleDrive` — archives the resume PDF into a “Qualified” folder.
- **Key configuration choices:**
  - File name: `CandidateName_Resume_YYYY-MM-DD.pdf` (spaces replaced with `_`)
  - Drive: “My Drive”
  - Folder: `qualified_resumes_folder_id` (placeholder)
- **Inputs / outputs:**
  - **Input:** from **IF: Qualified Candidate?** (true)
  - **Output:** to **Merge: Qualified Path** (input index 1)
- **Credentials:** Google Drive OAuth2 (`PDFMunk - Jitesh`)
- **Edge cases / failures:**
  - Requires binary PDF data to be present/mapped; not shown explicitly
  - Folder ID placeholders must be replaced
  - File name sanitization: only whitespace handled; other invalid filename characters may fail upload

#### Node: Slack: Qualified Alert
- **Type / role:** `n8n-nodes-base.slack` — posts a formatted Slack message to HR channel.
- **Key configuration choices:**
  - OAuth2 authentication
  - Channel ID uses environment variable: `{{$vars.SLACK_HR_CHANNEL_ID}}`
  - Message includes candidate summary, top skills, certifications, LinkedIn, score/tier
- **Inputs / outputs:**
  - **Input:** from **HubSpot: Create Contact**
  - **Output:** to **Merge: Qualified Path** (input index 0)
- **Edge cases / failures:**
  - Missing `SLACK_HR_CHANNEL_ID` variable
  - Slack OAuth scopes missing (chat:write)
  - Message formatting assumptions (arrays, lengths) fail if fields missing

#### Node: Google Drive: Archive Rejected
- **Type / role:** `n8n-nodes-base.googleDrive` — archives rejected resumes to a separate folder.
- **Key configuration choices:**
  - Same naming scheme as qualified
  - Folder: `rejected_resumes_folder_id` (placeholder)
- **Inputs / outputs:**
  - **Input:** from **IF: Qualified Candidate?** (false)
  - **Output:** to **Merge: Rejected Path** (input index 0)
- **Edge cases / failures:** same as Archive Qualified, plus ensuring distinct folder exists

#### Node: Gmail: Send Rejection
- **Type / role:** `n8n-nodes-base.gmail` — sends an HTML rejection email with personalized feedback.
- **Key configuration choices:**
  - To: `{{$json.candidateEmail}}`
  - Subject: `Thank you for your application - {{$json.candidateName}}`
  - HTML body uses:
    - First name: `{{$json.candidateName.split(' ')[0]}}`
    - Conditional feedback based on skillsByCategory.cloud and certifications length
- **Inputs / outputs:**
  - **Input:** from **IF: Qualified Candidate?** (false)
  - **Output:** to **Slack: Rejection Log**
- **Credentials:** Gmail OAuth2 (`jitesh0dugar@gmail.com`)
- **Edge cases / failures:**
  - If candidateEmail is default/fallback `user@example.com`, rejection may go to wrong address
  - candidateName split may fail if empty/non-string
  - Gmail API sending limits / “From” restrictions

#### Node: Slack: Rejection Log
- **Type / role:** `n8n-nodes-base.slack` — logs rejection outcome to Slack channel.
- **Key configuration choices:**
  - Channel ID: `{{$vars.SLACK_HR_CHANNEL_ID}}`
  - Message includes reason: “Score below 70 threshold”
- **Inputs / outputs:**
  - **Input:** from **Gmail: Send Rejection**
  - **Output:** to **Merge: Rejected Path** (input index 1)
- **Edge cases / failures:** same Slack issues as qualified alert

#### Node: Merge: Qualified Path
- **Type / role:** `n8n-nodes-base.merge` — joins the two qualified-side actions (Slack alert and Drive archive) before analytics.
- **Key configuration choices:** default merge behavior (as configured in UI; JSON shows empty parameters)
- **Inputs / outputs:**
  - **Inputs:** 
    - Input 0 from **Slack: Qualified Alert**
    - Input 1 from **Google Drive: Archive Qualified**
  - **Output:** to **Code: Analytics Calculator**
- **Edge cases / failures:**
  - Merge semantics matter: if one branch fails or doesn’t emit, merge may stall or drop items depending on mode (not specified)

#### Node: Merge: Rejected Path
- **Type / role:** `n8n-nodes-base.merge` — joins rejected-side actions (Drive archive and Slack log) before analytics.
- **Inputs / outputs:**
  - **Inputs:**
    - Input 0 from **Google Drive: Archive Rejected**
    - Input 1 from **Slack: Rejection Log**
  - **Output:** not connected directly; analytics node likely relies on both Merge nodes feeding it via separate connection paths (in this workflow only the qualified merge is connected onward; see “Connections” notes below).
- **Edge cases / failures:**
  - As wired, **Merge: Rejected Path does not feed analytics** (no outgoing connection in provided connections). That likely means rejected candidates never reach analytics storage unless there’s an implicit connection not shown (but JSON connections show none).

---

### Block D — Analytics & Feedback Loop (Phase 4)

**Overview:**  
Computes funnel metrics (processing time, skill match percentage, flags), then stores them in PostgreSQL.

**Nodes involved:**
- Code: Analytics Calculator
- PostgreSQL: Store Analytics

#### Node: Code: Analytics Calculator
- **Type / role:** `n8n-nodes-base.code` — aggregates per-candidate analytics metrics.
- **Key configuration choices:**
  - For each input item:
    - `processingTimeSeconds = analysisTimestamp - receivedDate`
    - `outcome = qualified ? "Accepted" : "Rejected"`
    - `skillMatchPercentage = totalSkills / 45 * 100`
    - Flags: hasLinkedIn, hasCertifications
    - `experienceLevel`: Senior (>=7), Mid-Level (>=3), else Junior
    - Adds `analytics.timestamp`
- **Inputs / outputs:**
  - **Input:** from **Merge: Qualified Path**
  - **Output:** to **PostgreSQL: Store Analytics**
- **Edge cases / failures:**
  - Date parsing issues if `receivedDate` is not ISO or missing
  - `certifications` assumed to exist and be an array

#### Node: PostgreSQL: Store Analytics
- **Type / role:** `n8n-nodes-base.postgres` — inserts analytics record into `hiring_funnel_analytics`.
- **Key configuration choices:**
  - Inserts outcome, score, tier, experience metrics, skill metrics, degree, flags, processing times, dates
- **Inputs / outputs:**
  - **Input:** from **Code: Analytics Calculator**
  - **Output:** none connected
- **Edge cases / failures:**
  - Same SQL interpolation/escaping risks as earlier insert
  - Schema expectations: numeric vs text columns must match

**Important wiring note (potential defect):**  
Based on the provided `connections`, only **qualified** items flow into **Code: Analytics Calculator**. **Merge: Rejected Path** has no outgoing connection, so rejected items do not reach analytics storage.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | n8n-nodes-base.stickyNote | High-level workflow documentation panel |  |  |  |
| Sticky Note - Intake | n8n-nodes-base.stickyNote | Visual phase label/comment |  |  |  |
| Sticky Note - Analysis | n8n-nodes-base.stickyNote | Visual phase label/comment |  |  |  |
| Sticky Note - Routing | n8n-nodes-base.stickyNote | Visual phase label/comment |  |  |  |
| Sticky Note - Analytics | n8n-nodes-base.stickyNote | Visual phase label/comment |  |  |  |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Poll Gmail for new emails | — | Code: Pre-Validation | ## 📧 PHASE 1: Intelligent Intake & Validation<br>Monitors Gmail for resume submissions. Validates PDF attachments and extracts raw text content for AI processing. Handles multiple attachment formats and sizes. |
| Code: Pre-Validation | n8n-nodes-base.code | Validate attachments, select PDF, metadata enrichment | Gmail Trigger | IF: Valid PDF Attachment? | ## 📧 PHASE 1: Intelligent Intake & Validation<br>Monitors Gmail for resume submissions. Validates PDF attachments and extracts raw text content for AI processing. Handles multiple attachment formats and sizes. |
| IF: Valid PDF Attachment? | n8n-nodes-base.if | Gate: attachment exists and first attachment is a PDF | Code: Pre-Validation | PDF to Text: Extract Content (true) | ## 📧 PHASE 1: Intelligent Intake & Validation<br>Monitors Gmail for resume submissions. Validates PDF attachments and extracts raw text content for AI processing. Handles multiple attachment formats and sizes. |
| PDF to Text: Extract Content | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Parse PDF into text/JSON | IF: Valid PDF Attachment? | Code: AI Resume Parser | ## 🧠 PHASE 2: AI-Powered Resume Analysis<br>Extracts structured data: contact info, skills, experience, education. Uses NLP pattern matching and scoring algorithms to calculate qualification metrics. Assigns candidate tier (A/B/C/D). |
| Code: AI Resume Parser | n8n-nodes-base.code | Extract candidate fields + compute score/tier | PDF to Text: Extract Content | PostgreSQL: Log Application; IF: Qualified Candidate? | ## 🧠 PHASE 2: AI-Powered Resume Analysis<br>Extracts structured data: contact info, skills, experience, education. Uses NLP pattern matching and scoring algorithms to calculate qualification metrics. Assigns candidate tier (A/B/C/D). |
| PostgreSQL: Log Application | n8n-nodes-base.postgres | Insert candidate application row | Code: AI Resume Parser | — | ## 🧠 PHASE 2: AI-Powered Resume Analysis<br>Extracts structured data: contact info, skills, experience, education. Uses NLP pattern matching and scoring algorithms to calculate qualification metrics. Assigns candidate tier (A/B/C/D). |
| IF: Qualified Candidate? | n8n-nodes-base.if | Branch on qualified threshold | Code: AI Resume Parser | (true) HubSpot: Create Contact, Google Drive: Archive Qualified; (false) Google Drive: Archive Rejected, Gmail: Send Rejection | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| HubSpot: Create Contact | n8n-nodes-base.hubspot | Create CRM contact | IF: Qualified Candidate? (true) | Slack: Qualified Alert | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| Slack: Qualified Alert | n8n-nodes-base.slack | Notify HR of qualified candidate | HubSpot: Create Contact | Merge: Qualified Path | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| Google Drive: Archive Qualified | n8n-nodes-base.googleDrive | Store qualified resume in Drive | IF: Qualified Candidate? (true) | Merge: Qualified Path | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| Google Drive: Archive Rejected | n8n-nodes-base.googleDrive | Store rejected resume in Drive | IF: Qualified Candidate? (false) | Merge: Rejected Path | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| Gmail: Send Rejection | n8n-nodes-base.gmail | Send rejection email with feedback | IF: Qualified Candidate? (false) | Slack: Rejection Log | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| Slack: Rejection Log | n8n-nodes-base.slack | Notify HR of rejection | Gmail: Send Rejection | Merge: Rejected Path | ## 🎯 PHASE 3: Smart Routing & Integration<br>Qualified candidates (70+ score) → HubSpot CRM + Slack alert + Drive archive. Unqualified candidates → Automated rejection email with personalized feedback. All actions logged to analytics. |
| Merge: Qualified Path | n8n-nodes-base.merge | Join Slack + Drive results (qualified path) | Slack: Qualified Alert; Google Drive: Archive Qualified | Code: Analytics Calculator | ## 📊 PHASE 4: Analytics & Feedback Loop<br>Tracks hiring funnel metrics, candidate quality trends, and time-to-hire. Feeds data back to scoring model for continuous improvement. |
| Merge: Rejected Path | n8n-nodes-base.merge | Join Slack + Drive results (rejected path) | Google Drive: Archive Rejected; Slack: Rejection Log | — | ## 📊 PHASE 4: Analytics & Feedback Loop<br>Tracks hiring funnel metrics, candidate quality trends, and time-to-hire. Feeds data back to scoring model for continuous improvement. |
| Code: Analytics Calculator | n8n-nodes-base.code | Compute funnel metrics per candidate | Merge: Qualified Path | PostgreSQL: Store Analytics | ## 📊 PHASE 4: Analytics & Feedback Loop<br>Tracks hiring funnel metrics, candidate quality trends, and time-to-hire. Feeds data back to scoring model for continuous improvement. |
| PostgreSQL: Store Analytics | n8n-nodes-base.postgres | Insert analytics row | Code: Analytics Calculator | — | ## 📊 PHASE 4: Analytics & Feedback Loop<br>Tracks hiring funnel metrics, candidate quality trends, and time-to-hire. Feeds data back to scoring model for continuous improvement. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the trigger**
   1. Add node: **Gmail Trigger**
   2. Set polling to **Every Minute**
   3. (Recommended) Add Gmail filters/label to only capture resume submissions
   4. Configure **Gmail OAuth2** credentials

2. **Add intake validation**
   1. Add node: **Code** → name it **Code: Pre-Validation**
   2. Paste logic that:
      - checks `attachments`
      - finds a PDF attachment
      - rejects >10MB
      - sets `pdfAttachment`, `applicantEmail`, `applicantName`, `receivedDate`, `fileSizeKB`, `skipProcessing`
   3. Connect: **Gmail Trigger → Code: Pre-Validation**

3. **Gate on PDF presence**
   1. Add node: **IF** → name it **IF: Valid PDF Attachment?**
   2. Add conditions:
      - `attachments` not empty
      - `attachments[0].mimeType` equals `application/pdf`
   3. Connect: **Code: Pre-Validation → IF: Valid PDF Attachment?**
   4. (Recommended fix) Change this IF to check the `pdfAttachment` field created by pre-validation, not `attachments[0]`.

4. **Parse PDF to text**
   1. Add node: **HTMLCSS to PDF** (the `htmlcsstopdf` community/partner node in your instance)
   2. Operation: **PDF Manipulation → Parse PDF to JSON**
   3. Configure credentials for the PDF parsing provider (`htmlcsstopdfApi`)
   4. Ensure the node is receiving the PDF binary/attachment correctly (map the selected attachment/binary if required by the node)
   5. Connect: **IF (true) → PDF to Text: Extract Content**

5. **Extract candidate data + score**
   1. Add node: **Code** → name it **Code: AI Resume Parser**
   2. Paste the parser/scoring script
   3. Confirm it reads from `item.json.text`
   4. Connect: **PDF to Text: Extract Content → Code: AI Resume Parser**

6. **Log raw application to PostgreSQL**
   1. Add node: **PostgreSQL** → name it **PostgreSQL: Log Application**
   2. Operation: **Execute Query**
   3. Create table `candidate_applications` with matching columns (email, name, phone, linkedin, location, skills json/text, etc.)
   4. Configure Postgres credentials (host/db/user/password/SSL as needed)
   5. Connect: **Code: AI Resume Parser → PostgreSQL: Log Application**
   6. (Recommended) Replace string-interpolated SQL with parameterized queries to avoid quote-breaking.

7. **Branch on qualification**
   1. Add node: **IF** → name it **IF: Qualified Candidate?**
   2. Condition: boolean equals `true` for `{{$json.qualified}}`
   3. Connect: **Code: AI Resume Parser → IF: Qualified Candidate?**

8. **Qualified path: HubSpot + Drive + Slack**
   1. Add node: **HubSpot** → **Create Contact**
      - Authentication: **App Token**
      - Map at least Email (`candidateEmail`) and Name (`candidateName`) in properties
   2. Add node: **Google Drive** → upload/archive
      - Drive: My Drive
      - Folder: create/select **Qualified** folder; paste folder ID
      - Name expression: `{{$json.candidateName.replace(/\s+/g, '_')}}_Resume_{{$now.toFormat('yyyy-MM-dd')}}.pdf`
      - Map the resume binary to upload
   3. Add node: **Slack** → post message (OAuth2)
      - Channel ID: set workflow/environment variable `SLACK_HR_CHANNEL_ID` or hardcode a channel
      - Paste the formatted message template
   4. Connect:
      - **IF (true) → HubSpot: Create Contact → Slack: Qualified Alert**
      - **IF (true) → Google Drive: Archive Qualified**

9. **Rejected path: Drive + Gmail + Slack**
   1. Add node: **Google Drive** → archive to **Rejected** folder
   2. Add node: **Gmail** → **Send**
      - To: `{{$json.candidateEmail}}`
      - Subject/body: use the provided HTML template and expressions
   3. Add node: **Slack** → rejection log message
   4. Connect:
      - **IF (false) → Google Drive: Archive Rejected**
      - **IF (false) → Gmail: Send Rejection → Slack: Rejection Log**

10. **Merge each branch**
   1. Add node: **Merge** → name it **Merge: Qualified Path**
      - Connect **Slack: Qualified Alert → Merge (Input 0)**
      - Connect **Drive: Archive Qualified → Merge (Input 1)**
   2. Add node: **Merge** → name it **Merge: Rejected Path**
      - Connect **Drive: Archive Rejected → Merge (Input 0)**
      - Connect **Slack: Rejection Log → Merge (Input 1)**

11. **Compute analytics and store**
   1. Add node: **Code** → name it **Code: Analytics Calculator**
      - Paste the analytics script
   2. Add node: **PostgreSQL** → name it **PostgreSQL: Store Analytics**
      - Operation: Execute Query
      - Create table `hiring_funnel_analytics` with required columns
   3. Connect:
      - **Merge: Qualified Path → Code: Analytics Calculator → PostgreSQL: Store Analytics**
      - (Recommended) Also connect **Merge: Rejected Path → Code: Analytics Calculator** so rejections are included (the provided workflow currently does not).

12. **Set required variables**
   - Add environment/workflow variable: `SLACK_HR_CHANNEL_ID`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ⚖️ Talent Sovereign: AI Resume Intelligence Hub — Industrial-grade recruitment pipeline: Gmail Intake → AI Parsing → Scoring → Smart Routing. | Sticky note “Documentation” (top-left) |
| Setup: Drive folders ‘Qualified’ and ‘Rejected’; Connect HubSpot App Token; Prepare `candidate_applications` table in Postgres. | Sticky note “Documentation” |
| Metrics tracked: `Qualification_Score`, `Skill_Match_%`, `Funnel_Conversion`. | Sticky note “Documentation” |

**Operational cautions (important):**
- The PDF validation gate checks `attachments[0]` and may drop valid PDFs that are not the first attachment; prefer checking the selected `pdfAttachment`.
- Both PostgreSQL nodes interpolate values directly into SQL strings; candidate names/emails containing quotes can break inserts. Parameterized queries (or proper escaping) are strongly recommended.
- Rejected-path merge is not connected to analytics in the provided wiring; connect it if you want complete funnel metrics.

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.