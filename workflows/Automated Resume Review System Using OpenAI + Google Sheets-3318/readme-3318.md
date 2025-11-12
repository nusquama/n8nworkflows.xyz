Automated Resume Review System Using OpenAI + Google Sheets

https://n8nworkflows.xyz/workflows/automated-resume-review-system-using-openai---google-sheets-3318


# Automated Resume Review System Using OpenAI + Google Sheets

### 1. Workflow Overview

This workflow automates the resume review process for HR teams, recruiters, and hiring managers by leveraging n8n automation and OpenAI’s AI capabilities. It streamlines intake, extraction, summarization, evaluation, and record-keeping of candidate resumes submitted via a form.

**Target Use Cases:**
- HR teams and recruitment agencies managing large applicant volumes.
- Hiring managers seeking quick candidate insights.
- Startups/small businesses automating hiring without complex ATS.
- AI professionals building smart HR workflows.

**Logical Blocks:**

- **1.1 Submission, Saving to Google Drive & Extraction:**  
  Captures resume submissions via an n8n form, uploads the resume PDF to Google Drive, and extracts raw text from the file.

- **1.2 Extraction (Personal Info & Qualification):**  
  Uses AI-powered information extraction to parse personal details (name, email, city, etc.) and professional qualifications (education, job history, skills) from the extracted resume text.

- **1.3 Merge & Summarization:**  
  Combines extracted data into a structured format and generates a concise professional summary of the candidate’s profile.

- **1.4 Voting, Consideration & Google Sheets Logging:**  
  An HR expert AI model reviews the summary against a desired candidate profile, assigns a rating and comments, then appends all data to a Google Sheet for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Submission, Saving to Google Drive & Extraction

**Overview:**  
This block handles user resume submission via a form, uploads the resume file to Google Drive, and extracts the text content from the PDF for further processing.

**Nodes Involved:**  
- On form submission  
- Upload to google drive  
- Resume extraction  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input (first name, last name, email, resume PDF) via a web form titled "Submit your CV".  
  - Config: Required fields for first name, last name, email, and PDF file upload. Ignores bot submissions.  
  - Input: User submits form.  
  - Output: Form data including uploaded file.  
  - Edge cases: Missing required fields, unsupported file types, bot submissions.  
  - Notes: Webhook ID used for external form integration.

- **Upload to google drive**  
  - Type: Google Drive node  
  - Role: Saves the uploaded resume PDF to Google Drive for storage and record-keeping.  
  - Config: File named dynamically with current date and original filename; uploaded to root folder of "My Drive".  
  - Input: File from form submission node.  
  - Output: Confirmation and metadata of uploaded file.  
  - Edge cases: Google Drive auth errors, quota limits, file upload failures.

- **Resume extraction**  
  - Type: Extract from File  
  - Role: Extracts raw text content from the uploaded PDF resume for AI processing.  
  - Config: Operation set to PDF extraction; binary property set to the uploaded resume file.  
  - Input: Uploaded PDF file.  
  - Output: Extracted text content of resume.  
  - Edge cases: Corrupted PDFs, unsupported PDF formats, extraction failures.

---

#### 2.2 Extraction (Personal Info & Qualification)

**Overview:**  
This block uses AI information extraction nodes to parse structured personal details and professional qualifications from the raw resume text.

**Nodes Involved:**  
- Personal Info  
- Qualification  

**Node Details:**

- **Personal Info**  
  - Type: LangChain Information Extractor  
  - Role: Extracts candidate’s personal details such as first name, last name, email, telephone, city, birthdate, LinkedIn, website, and summary from resume text.  
  - Config: Manual JSON schema defining expected fields; input text is the extracted resume text.  
  - Input: Text from Resume extraction node.  
  - Output: JSON object with personal info fields.  
  - Edge cases: Missing or ambiguous data in resume, extraction inaccuracies.

- **Qualification**  
  - Type: LangChain Information Extractor  
  - Role: Extracts educational qualifications, job history, and skills from resume text.  
  - Config: Uses a system prompt instructing to extract only relevant info; attributes defined with descriptions and required flags.  
  - Input: Text from Resume extraction node.  
  - Output: JSON object with education, job history, and skills (bulleted list).  
  - Edge cases: Partial or vague qualification info, inconsistent formatting.

---

#### 2.3 Merge & Summarization

**Overview:**  
Merges the extracted personal and qualification data into a single structured object, then generates a concise professional summary using AI summarization.

**Nodes Involved:**  
- Merge  
- Summarizer  
- wanted profile  

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Role: Combines outputs from Personal Info and Qualification nodes into one unified JSON object.  
  - Config: Mode set to "combine" with "combineAll" option to merge all incoming data.  
  - Input: Outputs from Personal Info and Qualification nodes.  
  - Output: Combined JSON with all extracted candidate data.  
  - Edge cases: Conflicting or missing fields, merge failures.

- **Summarizer**  
  - Type: LangChain Chain Summarization  
  - Role: Generates a concise, professional summary of the candidate’s profile based on merged data.  
  - Config: Custom prompt template including first name, last name, city, education, experience, skills, and applied date; summary limited to 100 words or less.  
  - Input: Merged candidate data.  
  - Output: Text summary of candidate qualifications.  
  - Edge cases: Incomplete data causing poor summaries, API timeouts.

- **wanted profile**  
  - Type: Set node  
  - Role: Defines the ideal candidate profile as a string for comparison by the HR expert AI.  
  - Config: Hardcoded description of desired skills and experience (automation expert with n8n, Python, JavaScript, etc.).  
  - Input: None (static).  
  - Output: JSON with "wanted-profile" string.  
  - Edge cases: Profile mismatch with actual candidates, requiring manual update for different roles.

---

#### 2.4 Voting, Consideration & Google Sheets Logging

**Overview:**  
An AI HR expert reviews the candidate summary against the desired profile, assigns a rating and consideration notes, then appends all data to a Google Sheet for record-keeping.

**Nodes Involved:**  
- HR Expert  
- Structured Output Parser  
- Google Sheets  

**Node Details:**

- **HR Expert**  
  - Type: LangChain Chain LLM  
  - Role: AI model acting as HR expert to rate candidate fit (1-10) and provide hiring considerations based on summary and desired profile.  
  - Config: Prompt instructs rating and explanation; input includes wanted profile and candidate summary; output parsed by Structured Output Parser.  
  - Input: wanted profile and Summarizer output.  
  - Output: JSON with "vote" (rating) and "consideration" (comments).  
  - Edge cases: AI misinterpretation, inconsistent ratings, API failures.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses HR Expert’s AI output into structured JSON fields "vote" and "consideration" for downstream use.  
  - Config: Manual JSON schema defining expected output fields.  
  - Input: HR Expert raw output.  
  - Output: Parsed JSON with rating and comments.  
  - Edge cases: Parsing errors if AI output deviates from schema.

- **Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends all extracted and evaluated candidate data into a centralized Google Sheet for tracking and filtering.  
  - Config: Maps multiple columns (first name, last name, city, DOB, email, skills, website, experience, education, vote, consideration, applied date) from merged data and AI outputs; appends to sheet "Sheet1" in specified Google Sheet document.  
  - Input: Merged data combined with HR Expert rating and consideration.  
  - Output: Confirmation of row append.  
  - Credentials: Google Sheets OAuth2 account configured.  
  - Edge cases: Authentication errors, quota limits, schema mismatches.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                                   | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                  |
|-------------------------|--------------------------------------|--------------------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission      | n8n Form Trigger                     | Captures resume submission via web form          | -                             | Upload to google drive, Resume extraction | ## Submission, Saving to Google Drive & Extraction Captures user info from the form. Uploads resume to Google Drive. Extracts data from the PDF (resume). |
| Upload to google drive  | Google Drive                        | Saves uploaded resume PDF to Google Drive        | On form submission            | Resume extraction           | ## Submission, Saving to Google Drive & Extraction Captures user info from the form. Uploads resume to Google Drive. Extracts data from the PDF (resume). |
| Resume extraction       | Extract from File                   | Extracts text from uploaded PDF resume            | Upload to google drive        | Personal Info, Qualification | ## Submission, Saving to Google Drive & Extraction Captures user info from the form. Uploads resume to Google Drive. Extracts data from the PDF (resume). |
| Personal Info           | LangChain Information Extractor    | Extracts personal details from resume text       | Resume extraction             | Merge                       | ## Extraction (Personal Info & Qualification) Extracts personal details (name, city, etc.). Retrieves educational qualifications and job history. |
| Qualification           | LangChain Information Extractor    | Extracts education, job history, and skills      | Resume extraction             | Merge                       | ## Extraction (Personal Info & Qualification) Extracts personal details (name, city, etc.). Retrieves educational qualifications and job history. |
| Merge                   | Merge                             | Combines personal info and qualification data    | Personal Info, Qualification  | Summarizer                  | ## Merge & Summarization Merges extracted information. Generates a concise professional summary.             |
| Summarizer              | LangChain Chain Summarization      | Generates concise candidate summary               | Merge                        | wanted profile              | ## Merge & Summarization Merges extracted information. Generates a concise professional summary.             |
| wanted profile          | Set                               | Defines desired candidate profile for evaluation | Summarizer                   | HR Expert                   | ## Merge & Summarization Merges extracted information. Generates a concise professional summary.             |
| HR Expert               | LangChain Chain LLM                | Rates candidate fit and provides hiring insights | wanted profile, Summarizer   | Structured Output Parser    | ## Voting, Consideration & Google Sheets HR expert reviews and analyzes the summary. Assigns rating and insights. Appends details to Google Sheets. |
| Structured Output Parser| LangChain Structured Output Parser | Parses HR Expert output into structured JSON     | HR Expert                   | Google Sheets               | ## Voting, Consideration & Google Sheets HR expert reviews and analyzes the summary. Assigns rating and insights. Appends details to Google Sheets. |
| Google Sheets           | Google Sheets                     | Logs all candidate data and evaluations           | Structured Output Parser, Merge | -                         | ## Voting, Consideration & Google Sheets HR expert reviews and analyzes the summary. Assigns rating and insights. Appends details to Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form titled "Submit your CV" with fields:  
     - First name (text, required)  
     - Last name (text, required)  
     - Email (email, required)  
     - Resume (file upload, required, accept PDF only)  
   - Enable "Ignore Bots" option.  
   - Save webhook URL for external form integration.

2. **Add Google Drive Node ("Upload to google drive")**  
   - Type: Google Drive  
   - Credential: Connect Google Drive OAuth2 account.  
   - Set file name: `Resume-{{ $now.format('yyyyLLdd') }}-{{ $json.Resume[0].filename }}`  
   - Upload to "My Drive" root folder.  
   - Input data field: "Resume" (from form submission).  
   - Connect output of form trigger to this node.

3. **Add Extract from File Node ("Resume extraction")**  
   - Type: Extract from File  
   - Operation: PDF  
   - Binary property name: "Resume" (from uploaded file).  
   - Connect output of Google Drive node to this node.

4. **Add LangChain Information Extractor Node ("Personal Info")**  
   - Type: LangChain Information Extractor  
   - Input text: `={{ $json.text }}` (text extracted from resume)  
   - Define manual JSON schema with fields: first_name, last_name, email, telephone, city, birthdate, linkedin, website, summary (all strings).  
   - Connect output of Resume extraction node to this node.

5. **Add LangChain Information Extractor Node ("Qualification")**  
   - Type: LangChain Information Extractor  
   - Input text: `={{ $json.text }}`  
   - System prompt: "You are an expert extraction algorithm. Only extract relevant information from the text. If you do not know the value of an attribute asked to extract, you may omit the attribute's value."  
   - Attributes to extract:  
     - Educational Qualification (required, max 100 words, include grades)  
     - Job History (required, max 100 words, recent experience)  
     - Skills (required, bulleted list of technical skills)  
   - Connect output of Resume extraction node to this node.

6. **Add Merge Node ("Merge")**  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Combine All  
   - Connect outputs of Personal Info and Qualification nodes to this node (Personal Info to input 2, Qualification to input 1).

7. **Add LangChain Chain Summarization Node ("Summarizer")**  
   - Type: LangChain Chain Summarization  
   - Prompt template:  
     ```
     Write a concise summary of the following:
     First name:{{ $json.output.first_name }}
     Last name:{{ $json.output.last_name }}
     City: {{ $json.output.city }}
     Educational Qualification:{{ $json.output['Educational Qualification'] }}
     Previous experience:{{ $json.output['Job History'] }}
     Skills:{{ $json.output.Skills }}
     Applied date:{{$now.format('yyyy-MM-dd')}}
     
     Write a concise Summary and summary of 100 words or less. Be concise and professional.
     ```  
   - Connect output of Merge node to this node.

8. **Add Set Node ("wanted profile")**  
   - Type: Set  
   - Add string field "wanted-profile" with the desired candidate profile text, e.g.:  
     ```
     We are a web agency looking for an Automation Expert skilled in workflow automation, API integrations, and AI-driven process optimization. The ideal candidate should have expertise in n8n, Python, and JavaScript, with a strong understanding of automation tools and webhooks. Experience in building custom automations for businesses is required. Requirements: Proficiency in n8n, Python, and JavaScript Experience in workflow automation, API integrations, and AI agents Ability to optimize business processes through automation Prior experience in the automation industry Must be based in Northern Italy If you have a passion for automation and want to work with a forward-thinking agency, we'd love to hear from you!
     ```  
   - Connect output of Summarizer node to this node.

9. **Add LangChain Chain LLM Node ("HR Expert")**  
   - Type: LangChain Chain LLM  
   - Prompt:  
     ```
     You are an HR expert, and your task is to determine whether a candidate aligns with the company's desired profile. You must assign a rating from 1 to 10, where 1 means the candidate does not meet the requirements, while 10 means the candidate is the perfect match for the role. Additionally, in the "consideration" field, explain the reasoning behind the given score.
     ```  
   - Input variables:  
     - Profile: `{{ $json['wanted-profile'] }}`  
     - Candidate: `{{ $('Summarizer').item.json.response.text }}`  
   - Enable output parser (to parse structured JSON).  
   - Connect output of wanted profile node to this node.

10. **Add LangChain Structured Output Parser Node ("Structured Output Parser")**  
    - Type: LangChain Structured Output Parser  
    - Define manual schema:  
      ```json
      {
        "type": "object",
        "properties": {
          "vote": { "type": "string" },
          "consideration": { "type": "string" }
        }
      }
      ```  
    - Connect output of HR Expert node to this node.

11. **Add Google Sheets Node ("Google Sheets")**  
    - Type: Google Sheets  
    - Credential: Connect Google Sheets OAuth2 account.  
    - Operation: Append  
    - Sheet: "Sheet1" (gid=0) in your target Google Sheet document.  
    - Map columns from merged data and AI outputs:  
      - First Name: `={{ $('Merge').item.json.output.first_name }}`  
      - Last Name: `={{ $('Merge').item.json.output.last_name }}`  
      - City: `={{ $('Merge').item.json.output.city }}`  
      - DOB: `={{ $('Merge').item.json.output.birthdate }}`  
      - Email: `={{ $('Merge').item.json.output.email }}`  
      - Skills: `={{ $('Merge').item.json.output.Skills }}`  
      - Website: `={{ $('Merge').item.json.output.website }}`  
      - Experience: `={{ $('Merge').item.json.output['Job History'] }}`  
      - Education Qualification: `={{ $('Merge').item.json.output['Educational Qualification'] }}`  
      - Vote: `={{ $json.output.vote }}` (from Structured Output Parser)  
      - Consideration: `={{ $json.output.consideration }}` (from Structured Output Parser)  
      - Applied Date: `={{ $now.format('MM-dd-yyyy') }}`  
    - Connect output of Structured Output Parser node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow streamlines resume submissions, extraction, summarization, and HR evaluation to improve hiring efficiency and consistency.                                                                                                                                                                                                                           | Workflow description and purpose.                                                                               |
| Customize the form trigger to use other form providers like Typeform or Tally if preferred.                                                                                                                                                                                                                                                                          | Form submission node.                                                                                            |
| Adjust the AI prompts in the Summarizer and HR Expert nodes to align with specific job requirements or company culture.                                                                                                                                                                                                                                            | Summarizer and HR Expert nodes.                                                                                  |
| Consider integrating Slack or email notifications to alert hiring managers when top candidates are processed.                                                                                                                                                                                                                                                       | Suggested workflow extension.                                                                                    |
| Connect this workflow to an ATS system to automate candidate pipeline updates.                                                                                                                                                                                                                                                                                      | Suggested workflow extension.                                                                                    |
| Google Drive and Google Sheets credentials require OAuth2 setup with appropriate scopes for file upload and sheet editing.                                                                                                                                                                                                                                         | Credential setup notes.                                                                                          |
| OpenAI API credentials must have access to GPT-4o-mini or similar models for best results.                                                                                                                                                                                                                                                                           | Credential setup notes.                                                                                          |
| The workflow includes sticky notes summarizing each block for easy understanding and maintenance.                                                                                                                                                                                                                                                                   | Sticky notes nodes in the workflow.                                                                              |
| For more information on n8n LangChain nodes and OpenAI integration, visit: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                                                                                                                                                                                                           | Official n8n documentation.                                                                                      |

---

This structured documentation provides a comprehensive understanding of the Automated Resume Review System workflow, enabling users and AI agents to analyze, reproduce, and customize it effectively.