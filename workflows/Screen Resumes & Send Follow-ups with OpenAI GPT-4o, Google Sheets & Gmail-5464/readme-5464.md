Screen Resumes & Send Follow-ups with OpenAI GPT-4o, Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/screen-resumes---send-follow-ups-with-openai-gpt-4o--google-sheets---gmail-5464


# Screen Resumes & Send Follow-ups with OpenAI GPT-4o, Google Sheets & Gmail

---

### 1. Workflow Overview

This workflow automates the screening of candidate resumes submitted via a form, evaluates them using OpenAI GPT-4o based on predefined job criteria and cultural fit questions, and then records the evaluation results in Google Sheets. Depending on the evaluation decision (Proceed or Reject), it sends customized follow-up emails via Gmail to candidates. The workflow is designed for HR teams needing a scalable, AI-powered resume screening and communication system.

Logical blocks:

- **1.1 Input Reception**: Captures resume uploads with candidate details via a web form.
- **1.2 Resume Parsing**: Extracts text content from the uploaded PDF resumes.
- **1.3 AI Evaluation**: Uses OpenAI GPT-4o with a Langchain Agent to score resumes against role-specific and cultural fit criteria, producing a structured JSON output.
- **1.4 Decision Classification**: Classifies AI evaluation output into “Accepted” or “Rejected” categories.
- **1.5 Data Recording**: Appends evaluation results to separate Google Sheets tabs (Accepted or Rejected).
- **1.6 Follow-up Messaging**: Sends personalized Gmail messages based on candidate evaluation outcomes.
- **1.7 Supporting Elements**: Sticky notes provide instructions and configuration guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Collects candidate details and resume PDF via a web form trigger.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Captures form data (resume file PDF, email, phone number, name) on user submission.  
  - Config: Requires resume upload (.pdf only), email, phone number, and name fields.  
  - Input: HTTP webhook request from form submission.  
  - Output: JSON with form fields and binary resume file.  
  - Edge cases: Missing required fields, invalid file format, large file size, webhook connectivity issues.  
  - Notes: Phone number field can be toggled optional per sticky note.

---

#### 1.2 Resume Parsing

**Overview:**  
Extracts text content from the uploaded resume PDF file for AI processing.

**Nodes Involved:**  
- Extracting CV data

**Node Details:**  
- **Extracting CV data**  
  - Type: Extract From File (PDF)  
  - Role: Converts PDF binary data from the form into plain text for further processing.  
  - Config: Operates on binary property "Resume" containing PDF.  
  - Input: Binary resume file from form submission node.  
  - Output: JSON with extracted text content.  
  - Edge cases: Corrupted or scanned PDFs, extraction failure, unsupported PDF features.  
  - Failure handling: Workflow continues despite extraction issues.

---

#### 1.3 AI Evaluation

**Overview:**  
Processes extracted resume text with an AI agent to score and evaluate the candidate against customized job and cultural criteria.

**Nodes Involved:**  
- Screening & Evaluating Resume's  
- OpenAI 4o  
- Structured Output Parser1  
- OpenAI Chat Model  

**Node Details:**  
- **Screening & Evaluating Resume's**  
  - Type: Langchain Agent (AI agent)  
  - Role: Receives extracted resume text, applies dynamic prompt with placeholders (e.g. COMPANY_NAME, ROLE_NAME, CRITERIA) and scores the candidate on two layers: technical match and cultural fit.  
  - Config: System prompt defines evaluation layers, scoring (0–10 per item), threshold logic for Proceed/Reject decision, and expects structured JSON output including name, summary, scores, decision, and justification.  
  - Input: Resume text from Extracting CV data; JSON schema example from Structured Output Parser1.  
  - Output: AI-generated structured JSON evaluation.  
  - Edge cases: AI model API rate limits, prompt formatting errors, unexpected AI outputs, missing placeholders, timeout.  
  - On error: set to continue with regular output (non-blocking failure mode).  

- **OpenAI 4o**  
  - Type: Langchain LLM Chat Node (GPT-4o)  
  - Role: Provides GPT-4o model access to the agent for resume evaluation.  
  - Config: Model set to "gpt-4o".  
  - Credentials: OpenAI API key required.  
  - Input: Prompt from Screening & Evaluating Resume's node.  
  - Output: AI text response.  

- **Structured Output Parser1**  
  - Type: Langchain Output Parser (Structured JSON)  
  - Role: Parses AI raw output into the defined JSON schema for downstream nodes.  
  - Config: JSON schema example enforces expected keys and data types.  
  - Input: Output from Screening & Evaluating Resume's AI agent.  
  - Output: Structured JSON data with candidate evaluation results.  
  - Edge cases: Parsing failures if AI output deviates from schema.  

- **OpenAI Chat Model**  
  - Type: Langchain LLM Chat Node (GPT-4.1-mini)  
  - Role: Classifies the decision output text ("Proceed" or "Reject") into categories for routing.  
  - Config: Model set to "gpt-4.1-mini".  
  - Credentials: OpenAI API key required.  
  - Input: Decision text from Structured Output Parser1.  
  - Output: Category classification ("Accepted" or "Rejected").  

---

#### 1.4 Decision Classification

**Overview:**  
Determines if candidate is accepted or rejected based on AI decision text.

**Nodes Involved:**  
- Text Classifier

**Node Details:**  
- **Text Classifier**  
  - Type: Langchain Text Classifier  
  - Role: Categorizes AI evaluation decision text into "Accepted" or "Rejected" for branching.  
  - Config: Categories defined as "Accepted" for "Proceed" and "Rejected" for "Reject".  
  - Input: Decision string from AI evaluation output.  
  - Output: Classification used to select Google Sheets append node.  
  - Edge cases: Misclassification if decision text is ambiguous or misspelled.

---

#### 1.5 Data Recording

**Overview:**  
Appends candidate evaluation data to the appropriate Google Sheets tab per decision.

**Nodes Involved:**  
- Google Sheets2 (Accepted)  
- Google Sheets (Rejected)

**Node Details:**  
- **Google Sheets2 (Accepted)**  
  - Type: Google Sheets Node  
  - Role: Appends candidate data to "Accepted" sheet tab.  
  - Config: Maps structured AI output fields and form data (name, email, phone, summary, justification, criteria scores) to sheet columns.  
  - Sheet Name: "Accepted" (gid=0)  
  - Credentials: OAuth2 Google Sheets account.  
  - Input: Candidate evaluation JSON from Text Classifier node.  
  - Output: Confirmation of append operation.  
  - Edge cases: API auth failures, sheet access issues, malformed data.  

- **Google Sheets (Rejected)**  
  - Type: Google Sheets Node  
  - Role: Appends candidate data to "Rejected" sheet tab.  
  - Config: Same mapping as Accepted node but to "Rejected" sheet tab.  
  - Sheet Name: "Rejected" (gid=839612147)  
  - Credentials: OAuth2 Google Sheets account.  
  - Input: Candidate evaluation JSON from Text Classifier node.  
  - Output: Confirmation of append operation.  
  - Edge cases: Same as above.

---

#### 1.6 Follow-up Messaging

**Overview:**  
Sends personalized emails to candidates based on their evaluation decision.

**Nodes Involved:**  
- Gmail3 (Accepted)  
- Gmail (Rejected)

**Node Details:**  
- **Gmail3 (Accepted)**  
  - Type: Gmail Node  
  - Role: Sends interview invitation email to candidates who passed screening.  
  - Config: Email addressed to candidate's email with personal name; includes scheduling link and instructions.  
  - Parameters: Sender name "HR Dept", subject "Hiring Process".  
  - Credentials: OAuth2 Gmail account.  
  - Input: Candidate data from Google Sheets2 (Accepted).  
  - Output: Email send confirmation.  
  - Edge cases: Gmail API limits, invalid email addresses, connection errors.  

- **Gmail (Rejected)**  
  - Type: Gmail Node  
  - Role: Sends rejection email to candidates who did not pass screening.  
  - Config: Polite rejection message with justification note, addressed personally.  
  - Parameters: Subject "Hiring Process".  
  - Credentials: OAuth2 Gmail account.  
  - Input: Candidate data from Google Sheets (Rejected).  
  - Output: Email send confirmation.  
  - Edge cases: Same as above.

---

#### 1.7 Supporting Elements

**Overview:**  
Sticky notes provide setup instructions, usage tips, and configuration guidance for users.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**  
- **Sticky Note**  
  - Content: Notes that the input includes PDF resume file, name, and email; phone can be optional.  
- **Sticky Note1**  
  - Content: Detailed setup instructions for AI Resume Evaluator including prompt customization, scoring criteria, workflow steps, and tips.  
- **Sticky Note2**  
  - Content: Google Sheets setup instructions—create sheets named "Accepted" and "Rejected" with correct headers.  
- **Sticky Note3**  
  - Content: Gmail API connection and message customization instructions.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                        | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                           |
|------------------------------|----------------------------------|-------------------------------------|-----------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission            | Form Trigger                     | Capture user resume and details     | -                           | Extracting CV data                | **Input:** First the user will submit pdf file of his resume, name and email, phone optional          |
| Extracting CV data            | Extract From File (PDF)           | Extract text content from PDF       | On form submission           | Screening & Evaluating Resume's  |                                                                                                     |
| Screening & Evaluating Resume's | Langchain Agent (AI)             | Evaluate resume against criteria    | Extracting CV data, Structured Output Parser1, OpenAI 4o | Text Classifier                     | ✅ AI Resume Evaluator – Setup Instructions: prompt customization, scoring, workflow explanation     |
| OpenAI 4o                    | Langchain LLM Chat (GPT-4o)      | Provide AI model for evaluation     | Screening & Evaluating Resume's | Screening & Evaluating Resume's |                                                                                                     |
| Structured Output Parser1     | Langchain Output Parser (JSON)   | Parse AI output to structured JSON | Screening & Evaluating Resume's | Screening & Evaluating Resume's  |                                                                                                     |
| Text Classifier              | Langchain Text Classifier         | Classify decision (Accepted/Rejected)| Screening & Evaluating Resume's | Google Sheets2, Google Sheets    |                                                                                                     |
| Google Sheets2               | Google Sheets Node                | Append accepted candidates data     | Text Classifier              | Gmail3                          | **Google Sheet:** Connect google, create sheets "Accepted" and "Rejected" with headers               |
| Google Sheets                | Google Sheets Node                | Append rejected candidates data     | Text Classifier              | Gmail                           | **Google Sheet:** Connect google, create sheets "Accepted" and "Rejected" with headers               |
| Gmail3                      | Gmail Node                      | Send acceptance/follow-up email     | Google Sheets2               | -                                | ## Gmail: Connect Gmail API, customize message and subject                                           |
| Gmail                       | Gmail Node                      | Send rejection email                 | Google Sheets                | -                                | ## Gmail: Connect Gmail API, customize message and subject                                           |
| Sticky Note                 | Sticky Note                      | Setup instructions (Input)           | -                           | -                                | First the user will submit pdf file of his resume, name and email, phone optional                     |
| Sticky Note1                | Sticky Note                      | Setup instructions (AI Evaluator)    | -                           | -                                | AI Resume Evaluator setup instructions, prompt editing, scoring, workflow explanation                |
| Sticky Note2                | Sticky Note                      | Setup instructions (Google Sheets)   | -                           | -                                | Google Sheets setup: create "Accepted" and "Rejected" sheets with headers                            |
| Sticky Note3                | Sticky Note                      | Setup instructions (Gmail)            | -                           | -                                | Gmail API connection and message customization instructions                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission"):**  
   - Node type: Form Trigger  
   - Configure form titled "Please Upload your resume" with fields:  
     - Resume (file upload, accept .pdf only, required)  
     - Email (email, required)  
     - Phone Number (number, required or optional)  
     - Name (text, required)  
   - Save and activate webhook.

2. **Add Extract From File node ("Extracting CV data"):**  
   - Node type: Extract From File  
   - Set operation to "pdf"  
   - Use binary property "Resume" from form trigger as input.

3. **Add Langchain Agent node ("Screening & Evaluating Resume's"):**  
   - Node type: Langchain agent  
   - Configure prompt with placeholders for COMPANY_NAME, ROLE_NAME, ROLE_DESCRIPTION, CRITERIA 1-5, Q1-Q5, THRESHOLD.  
   - System message to instruct evaluation on two layers with scoring (0-10 per criteria), combined threshold for decision.  
   - Enable output parser expecting structured JSON with candidate info, scores, decision, and justification.  
   - Set onError to continue regular output.  
   - Connect input text to extracted resume text.

4. **Add OpenAI Chat node ("OpenAI 4o"):**  
   - Node type: Langchain LLM Chat  
   - Select model "gpt-4o"  
   - Connect as AI LLM for the agent node.  
   - Set OpenAI credentials (API key).

5. **Add Structured Output Parser node ("Structured Output Parser1"):**  
   - Node type: Langchain Output Parser (Structured)  
   - Provide JSON schema example matching expected AI output fields (name, summary, match_score, decision, justification, detailed job_match and cultural fit).  
   - Connect AI agent output to this parser.

6. **Add Langchain Chat node ("OpenAI Chat Model") for classification:**  
   - Node type: Langchain LLM Chat  
   - Use model "gpt-4.1-mini"  
   - Purpose: classify AI decision text into categories.  
   - Set OpenAI credentials.

7. **Add Langchain Text Classifier node ("Text Classifier"):**  
   - Node type: Langchain Text Classifier  
   - Categories:  
     - "Accepted" for decision "Proceed"  
     - "Rejected" for decision "Reject"  
   - Input text: AI decision from structured parser or chat model.

8. **Add two Google Sheets nodes:**  
   - Node 1 ("Google Sheets2" for Accepted):  
     - Operation: Append row  
     - Document: Connect to Google Sheets document by ID  
     - Sheet: "Accepted" tab (gid=0)  
     - Map columns: Candidate name, email (toLowerCase), phone, justification, summary, and criteria scores from AI output.  
     - Set Google Sheets OAuth2 credentials.  
   - Node 2 ("Google Sheets" for Rejected):  
     - Same as above but sheet tab "Rejected" (gid=839612147).

9. **Add two Gmail nodes for follow-up emails:**  
   - Node 1 ("Gmail3" for Accepted):  
     - Send to candidate email (toLowerCase)  
     - Subject: "Hiring Process"  
     - Message: Personalized acceptance email with interview scheduling link and instructions.  
     - Credentials: Gmail OAuth2.  
   - Node 2 ("Gmail" for Rejected):  
     - Send to candidate email (toLowerCase)  
     - Subject: "Hiring Process"  
     - Message: Polite rejection email with justification note.  
     - Credentials: Gmail OAuth2.

10. **Connect nodes in sequence:**  
    - On form submission → Extracting CV data → Screening & Evaluating Resume's → Structured Output Parser1 → OpenAI Chat Model → Text Classifier →  
      → If Accepted → Google Sheets2 → Gmail3  
      → If Rejected → Google Sheets → Gmail

11. **Add sticky notes for documentation:**  
    - Input instructions  
    - AI Evaluator setup guidance  
    - Google Sheets setup instructions  
    - Gmail setup and customization notes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| ✅ AI Resume Evaluator – Setup Instructions: Customize prompt with COMPANY NAME, ROLE NAME, ROLE DESCRIPTION, criteria, questions, and threshold. Workflow scores resumes and decides Proceed or Reject with justification.                                                                                                                                                                                   | Sticky Note1 content inside workflow                                                               |
| **Google Sheet**: Connect Google Sheets account. Create a document with two sheets named "Accepted" and "Rejected" with headers matching mapped columns.                                                                                                                                                                                                                                                 | Sticky Note2 content inside workflow                                                               |
| ## Gmail: Connect Gmail API OAuth2. Customize message body and subject lines for acceptance and rejection emails.                                                                                                                                                                                                                                                                                          | Sticky Note3 content inside workflow                                                               |
| Phone number field in form can be switched optional depending on use case.                                                                                                                                                                                                                                                                                                                                 | Sticky Note content inside workflow                                                                |
| Interview scheduling link used in acceptance email: https://cal.com/abdulaziz-saeed-lpurih/job-interview                                                                                                                                                                                                                                                                                                   | Gmail3 node email message content                                                                  |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---