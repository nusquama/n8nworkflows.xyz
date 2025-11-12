Automate Job Search with LinkedIn, Google Sheets & AI Resume Scoring & Cover Letters

https://n8nworkflows.xyz/workflows/automate-job-search-with-linkedin--google-sheets---ai-resume-scoring---cover-letters-9602


# Automate Job Search with LinkedIn, Google Sheets & AI Resume Scoring & Cover Letters

### 1. Workflow Overview

This workflow automates the entire job search process by integrating LinkedIn job listings, Google Sheets for job management, and AI-powered resume scoring and cover letter generation. It is designed for job seekers who want to efficiently find relevant job openings posted in the last 24 hours, analyze how well their resume fits each listing, generate tailored cover letters, and track all results in a Google Sheet, with notifications sent upon completion.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Input Preparation:** Initiates the workflow daily, downloads the user’s resume from Google Drive, extracts its text, and retrieves job search criteria from Google Sheets.

- **1.2 LinkedIn Job Search URL Construction & Job Fetching:** Constructs a LinkedIn job search URL based on criteria, fetches recent job postings from LinkedIn, and parses job listing URLs.

- **1.3 Job Details Retrieval & Parsing:** Iterates over each job URL to fetch detailed job information including title, company, location, description, and job ID.

- **1.4 AI Resume Analysis, Scoring & Cover Letter Generation:** Uses Google Gemini (PaLM) AI models to analyze job descriptions against the user’s resume, score job fit, identify resume improvement points, and generate personalized cover letters.

- **1.5 Results Storage & Notification:** Appends or updates the job search results with scores and cover letters in a Google Sheet and sends an email notification when the workflow completes.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Input Preparation

- **Overview:**  
  This initial block triggers the workflow once daily at 5 AM. It downloads the user’s resume from Google Drive and extracts its text content. It then retrieves job search criteria from a configured Google Sheet.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Download file (Google Drive)  
  - Extract from File  
  - Get row(s) in sheet (Google Sheets)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow every day at 5 AM.  
    - *Configuration:* Interval with trigger at hour 5 (5 AM daily).  
    - *Input/Output:* No input; output triggers next node.  
    - *Failure cases:* Workflow won't start if n8n is offline or misconfigured.

  - **Download file**  
    - *Type:* Google Drive Node (download operation)  
    - *Role:* Downloads the user’s resume file from Google Drive.  
    - *Configuration:* File ID expected to be set via URL mode; requires Google Drive OAuth2 credentials.  
    - *Input:* Trigger from Schedule Trigger node.  
    - *Output:* Binary file data passed to the next node.  
    - *Failure cases:* Invalid file ID, permission denied, expired credentials.

  - **Extract from File**  
    - *Type:* Extract from File (PDF operation)  
    - *Role:* Extracts text content from the downloaded PDF resume.  
    - *Configuration:* Operation set to "pdf".  
    - *Input:* Binary PDF file from Download file node.  
    - *Output:* Extracted text as JSON property `text`.  
    - *Failure cases:* Corrupted PDF, unsupported format, extraction errors.

  - **Get row(s) in sheet**  
    - *Type:* Google Sheets (read rows)  
    - *Role:* Retrieves job search filter criteria (keywords, location, experience level, etc.) from a Google Sheet.  
    - *Configuration:* Document ID and sheet name configured; requires Google Sheets OAuth2 credentials.  
    - *Input:* Trigger from Extract from File node.  
    - *Output:* JSON rows with job search parameters.  
    - *Failure cases:* Sheet access denied, invalid sheet ID, empty or malformed data.

---

#### 1.2 LinkedIn Job Search URL Construction & Job Fetching

- **Overview:**  
  Constructs a LinkedIn job search URL dynamically based on the retrieved criteria, then fetches the resulting job listings page HTML and extracts individual job posting URLs.

- **Nodes Involved:**  
  - LinkedIn Search URL (Code)  
  - Fetch jobs from LinkedIn (HTTP Request)  
  - HTML (HTML Extract)  
  - Split Out (Split Out)

- **Node Details:**

  - **LinkedIn Search URL**  
    - *Type:* Code (JavaScript)  
    - *Role:* Builds a LinkedIn job search URL with filters for keyword, location, experience level, remote options, and easy apply flag.  
    - *Configuration:* Uses input JSON fields such as `Keyword`, `Location`, `Experience Level`, `Remote`, `Easy Apply` from the Google Sheets data.  
    - *Key expressions:* String concatenation with LinkedIn URL parameters; experience and remote options mapped to LinkedIn codes (e.g., Internship = 1).  
    - *Input:* Job search criteria JSON from Google Sheets node.  
    - *Output:* JSON object with constructed `url` property.  
    - *Failure cases:* Missing or malformed input fields, incorrect mapping logic.

  - **Fetch jobs from LinkedIn**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves the LinkedIn job search results page HTML by requesting the constructed URL.  
    - *Configuration:* URL set dynamically from previous node's `url` property.  
    - *Input:* URL from LinkedIn Search URL node.  
    - *Output:* HTML content of LinkedIn job search results page.  
    - *Failure cases:* HTTP errors (e.g., 403, 429 rate limiting), LinkedIn blocking automated requests.

  - **HTML**  
    - *Type:* HTML Extract  
    - *Role:* Parses the fetched HTML to extract all job posting URLs (`href` attributes) from the job list.  
    - *Configuration:* CSS selector targets job cards anchor tags; extracts `href` attribute into an array under key `jobs`.  
    - *Input:* HTML from Fetch jobs from LinkedIn node.  
    - *Output:* JSON array of job URLs.  
    - *Failure cases:* HTML structure changes, empty or missing job listings.

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits the array of job URLs into individual items for further processing.  
    - *Configuration:* Field to split out set to `jobs`.  
    - *Input:* JSON array from HTML node.  
    - *Output:* Individual job URL items.  
    - *Failure cases:* Empty array input.

---

#### 1.3 Job Details Retrieval & Parsing

- **Overview:**  
  Iterates over each extracted LinkedIn job URL to fetch detailed job page content and parse key job information such as title, company, location, description, and job ID.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches)  
  - Wait  
  - HTTP Request  
  - HTML1 (HTML Extract)  
  - Edit Fields (Set)

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes job URLs in batches (default batch size) to control request pacing and avoid rate limits.  
    - *Configuration:* Default parameters; handles batch iteration.  
    - *Input:* Individual job URL items from Split Out node.  
    - *Output:* Passes each job URL sequentially for detailed processing.  
    - *Failure cases:* Large number of jobs may slow workflow; batch size tuning may be needed.

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Adds a 2-second delay between requests to avoid rate limiting or server blocking.  
    - *Configuration:* Wait amount set to 2 seconds.  
    - *Input:* Triggered after sending each message and before next HTTP request.  
    - *Output:* Delayed trigger.  
    - *Failure cases:* None significant; just increases runtime.

  - **HTTP Request**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the HTML content of each individual LinkedIn job posting page.  
    - *Configuration:* URL set dynamically from current job URL item.  
    - *Input:* Job URL from Loop Over Items.  
    - *Output:* Job posting HTML content.  
    - *Failure cases:* HTTP errors, blocked requests, missing pages.

  - **HTML1**  
    - *Type:* HTML Extract  
    - *Role:* Parses job page HTML to extract job title, company, location, job description, and job ID.  
    - *Configuration:* Multiple CSS selectors targeting LinkedIn job page elements; extracts text and attributes.  
    - *Input:* HTML from HTTP Request node.  
    - *Output:* JSON fields with job details.  
    - *Failure cases:* Page layout changes, missing fields, inconsistent selectors.

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Cleans and formats extracted job fields: trims whitespace in description, extracts last part of job ID, constructs apply link URL, copies other fields as-is.  
    - *Configuration:* Uses JavaScript expressions to clean and transform data.  
    - *Input:* JSON from HTML1 node.  
    - *Output:* Cleaned and enriched job data.  
    - *Failure cases:* Unexpected field formats, expression errors.

---

#### 1.4 AI Resume Analysis, Scoring & Cover Letter Generation

- **Overview:**  
  Uses AI models (Google Gemini via LangChain agent nodes) to analyze job description vs. extracted resume text, generate a detailed job fit JSON with scoring, produce a list of resume improvements, and create a tailored cover letter.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - Google Gemini Chat Model (AI model node)  
  - Edit Fields1 (Set)  
  - AI Agent1 (LangChain Agent)  
  - Google Gemini Chat Model1 (AI model node)

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent (AI analysis)  
    - *Role:* Receives job description and extracted resume text, parses into structured job analysis and resume analysis, scores fit, explains score, identifies gaps & suggestions, and drafts a cover letter per job.  
    - *Configuration:* Custom prompt instructing detailed JSON output with strict schema and content rules.  
    - *Input:* Job description (from Edit Fields) and resume text (from Extract from File).  
    - *Output:* Large JSON object with detailed analyses and cover letter as a single string.  
    - *Failure cases:* API errors, prompt misformatting, incomplete JSON output.

  - **Google Gemini Chat Model**  
    - *Type:* AI Language Model (Google PaLM Gemini)  
    - *Role:* Underlying AI model called by AI Agent node for processing.  
    - *Configuration:* Requires Google Palm API credentials.  
    - *Input/Output:* AI Agent delegates to this node transparently.  
    - *Failure cases:* API quota exceeded, invalid API key.

  - **Edit Fields1**  
    - *Type:* Set  
    - *Role:* Cleans the raw AI output string from AI Agent by stripping JSON code fences, normalizing quotes, trimming at `END_OF_JSON`, and parsing valid JSON object.  
    - *Configuration:* Uses JavaScript function to parse the AI response safely.  
    - *Input:* Raw JSON output string from AI Agent node.  
    - *Output:* Parsed JSON object with job_analysis, resume_analysis, scores, cover letter, etc.  
    - *Failure cases:* Malformed AI output, parse errors fallback to empty object.

  - **AI Agent1**  
    - *Type:* LangChain Agent  
    - *Role:* Takes the job description and resume again to produce a concise, prioritized list of resume editing suggestions to improve job fit.  
    - *Configuration:* Custom prompt specifying a strict numbered list of single-line points tagged by action type.  
    - *Input:* Job description and resume text.  
    - *Output:* String list of resume improvement points.  
    - *Failure cases:* API errors, prompt failures.

  - **Google Gemini Chat Model1**  
    - *Type:* AI Language Model (Google PaLM Gemini)  
    - *Role:* Underlying AI model called by AI Agent1.  
    - *Configuration:* Requires Google Palm API credentials.  
    - *Failure cases:* Same as above.

---

#### 1.5 Results Storage & Notification

- **Overview:**  
  Stores the processed job data, match scores, cover letters, and improvement suggestions into a Google Sheet and sends an email notification once all jobs are processed.

- **Nodes Involved:**  
  - Append or update row in sheet1 (Google Sheets)  
  - Loop Over Items (Split In Batches) [reused]  
  - Send a message (Gmail)  

- **Node Details:**

  - **Append or update row in sheet1**  
    - *Type:* Google Sheets (append or update)  
    - *Role:* Writes job data including apply link, match score, core skills, location, company, cover letter, and improvement suggestions into a result sheet. Uses the job apply link as unique key to update existing rows.  
    - *Configuration:* Document ID and sheet name set; columns mapped explicitly; requires Google Sheets OAuth2 credentials.  
    - *Input:* JSON data including job and AI analysis results.  
    - *Output:* Confirmation of row append/update.  
    - *Failure cases:* Sheet access denied, invalid sheet structure, large data size.

  - **Loop Over Items** (reused)  
    - *Role:* Reused to control batch processing of appending rows and sending messages.

  - **Send a message**  
    - *Type:* Gmail  
    - *Role:* Sends a notification email to the configured email address informing the user that the job search results with resume suggestions are ready in the Google Sheet.  
    - *Configuration:* Uses Gmail OAuth2 credentials; email subject and text message are preset.  
    - *Input:* Triggered after batch processing completes.  
    - *Failure cases:* Authentication failures, email quota exceeded, incorrect email address.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                                         | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                         |
|---------------------------|-------------------------------|--------------------------------------------------------|------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger               | Starts the workflow daily at 5 AM                       |                              | Download file                 |                                                                                                   |
| Download file             | Google Drive                  | Downloads the user's resume PDF                          | Schedule Trigger             | Extract from File             | ## Add Google Cloud Credentials 1. sign in with google 2. Upload your resume and change to anyone can view 3. Paste your link |
| Extract from File          | Extract from File             | Extracts text from PDF resume                            | Download file                | Get row(s) in sheet          |                                                                                                   |
| Get row(s) in sheet        | Google Sheets                 | Retrieves job search criteria from Google Sheet         | Extract from File            | LinkedIn Search URL           | ## Add Google Cloud Credentials 1. sign in with google 2. Prepare your sheet filter 3. Add your sheet Link |
| LinkedIn Search URL        | Code                         | Builds LinkedIn search URL from criteria                 | Get row(s) in sheet          | Fetch jobs from LinkedIn      |                                                                                                   |
| Fetch jobs from LinkedIn   | HTTP Request                 | Fetches LinkedIn job search results HTML                 | LinkedIn Search URL          | HTML                         |                                                                                                   |
| HTML                      | HTML Extract                 | Extracts job posting URLs from search results            | Fetch jobs from LinkedIn     | Split Out                    |                                                                                                   |
| Split Out                 | Split Out                    | Splits job URLs array into individual items              | HTML                        | Loop Over Items              |                                                                                                   |
| Loop Over Items           | Split In Batches             | Processes job URLs in batches                             | Split Out, Append or update row in sheet1 | Send a message, Wait      |                                                                                                   |
| Wait                      | Wait                         | Adds 2 seconds delay between requests                     | Loop Over Items              | HTTP Request                 |                                                                                                   |
| HTTP Request              | HTTP Request                 | Fetches individual job posting HTML                       | Wait                        | HTML1                        |                                                                                                   |
| HTML1                     | HTML Extract                 | Extracts detailed job info from job posting page          | HTTP Request                | Edit Fields                  |                                                                                                   |
| Edit Fields               | Set                          | Cleans and formats job info                               | HTML1                       | AI Agent                    |                                                                                                   |
| AI Agent                  | LangChain Agent              | Analyzes job vs. resume, scores fit, generates cover letter | Edit Fields                 | Edit Fields1                 | ## Add Gemini Free tier api key 1. sign in with google 2. Create API key and project 3. Add Free Tier key |
| Google Gemini Chat Model  | AI Language Model (Google Gemini) | Provides AI processing for AI Agent                       | AI Agent                    | AI Agent                    | ## Add Gemini Free tier api key 1. sign in with google 2. Create API key and project 3. Add Free Tier key |
| Edit Fields1              | Set                          | Parses and cleans AI Agent JSON output                    | AI Agent                    | AI Agent1                   |                                                                                                   |
| AI Agent1                 | LangChain Agent              | Generates resume improvement suggestions                   | Edit Fields1                | Append or update row in sheet1 |                                                                                                   |
| Google Gemini Chat Model1 | AI Language Model (Google Gemini) | Provides AI processing for AI Agent1                      | AI Agent1                   | AI Agent1                   |                                                                                                   |
| Append or update row in sheet1 | Google Sheets                 | Stores job data and AI results in Google Sheet            | AI Agent1                   | Loop Over Items             | ## Add Google Cloud Credentials 1. sign in with google 2. Select your sheet and sub sheet          |
| Send a message            | Gmail                        | Sends email notification of job search completion         | Loop Over Items             |                               | ## Add Google Cloud Credentials - sign in with google - Enter your e-mail address                |
| Sticky Note               | Sticky Note                  | Instructional notes for credentials and usage             |                              |                               | # Job search ultimate workflow - @jugaldb 1. Takes about 1 hour to run 2. Finds relevant jobs posted in last one day, runs at 5 AM everyday 3. Sends you an e-mail when complete |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 5 AM.

2. **Add Google Drive node ("Download file"):**  
   - Operation: Download  
   - Configure OAuth2 credentials for Google Drive.  
   - Set fileId to the URL or ID of your resume file shared with 'anyone can view'.

3. **Add Extract from File node:**  
   - Operation: PDF extraction.  
   - Connect input from "Download file".  
   - Outputs extracted resume text.

4. **Add Google Sheets node ("Get row(s) in sheet"):**  
   - Operation: Get rows.  
   - Set Document ID to your Google Sheet containing job search filters.  
   - Set Sheet Name to the relevant filter sheet (e.g., "Filter").  
   - Configure Google Sheets OAuth2 credentials.

5. **Add Code node ("LinkedIn Search URL"):**  
   - Use JavaScript code to build LinkedIn job search URL dynamically from incoming filter data.  
   - Map experience levels and remote work options to LinkedIn codes.  
   - Output: JSON object with `url` property.

6. **Add HTTP Request node ("Fetch jobs from LinkedIn"):**  
   - URL: Set dynamically from previous node’s `url`.  
   - Method: GET.

7. **Add HTML Extract node ("HTML"):**  
   - Operation: Extract HTML content.  
   - CSS Selector: `ul.jobs-search__results-list li div a[class*="base-card"]`  
   - Extract `href` attribute as array named `jobs`.

8. **Add Split Out node ("Split Out"):**  
   - Field to split out: `jobs`.

9. **Add Split In Batches node ("Loop Over Items"):**  
   - Default batch size (tune if needed).  
   - Input: From "Split Out" node.

10. **Add Wait node ("Wait"):**  
    - Wait time: 2 seconds.  
    - Connect from "Loop Over Items" (for pacing).

11. **Add HTTP Request node ("HTTP Request"):**  
    - URL: Set dynamically from current job URL.  
    - Method: GET.

12. **Add HTML Extract node ("HTML1"):**  
    - Extract job fields: Title, Company, Location, Description, Job ID.  
    - Use appropriate CSS selectors as in original.

13. **Add Set node ("Edit Fields"):**  
    - Clean Description by replacing multiple whitespace with a single space.  
    - Extract last segment of Job ID.  
    - Construct Apply Link URL by appending Job ID to LinkedIn jobs view URL.  
    - Pass other fields (Title, Company, Location) as is.

14. **Add LangChain Agent node ("AI Agent"):**  
    - Prompt: Use detailed schema and instructions for job-resume analysis, scoring, explanation, gaps, and cover letter generation.  
    - Inputs: Job Description (from "Edit Fields") and Resume Text (from "Extract from File").  
    - Connect AI node to Google Gemini Chat Model node for execution.  
    - Configure Google Palm API credentials for Gemini model.

15. **Add Set node ("Edit Fields1"):**  
    - Use JavaScript code to clean AI Agent output: strip JSON fences, normalize quotes, truncate at `END_OF_JSON`, parse JSON.  
    - Input: Raw output from AI Agent node.

16. **Add LangChain Agent node ("AI Agent1"):**  
    - Prompt: Generate crisp, prioritized resume editing suggestions.  
    - Inputs: Job Description and Resume Text.  
    - Connect to Google Gemini Chat Model1 node.  
    - Use same Google Palm API credentials.

17. **Add Google Sheets node ("Append or update row in sheet1"):**  
    - Operation: Append or update rows in result sheet.  
    - Map columns: Apply Link, Score, Title, Skills, Location, Company, Cover Letter, Improvements.  
    - Use Apply Link as unique key for updating rows.  
    - Configure Google Sheets OAuth2 credentials.  
    - Connect input from "AI Agent1" output.

18. **Connect "Append or update row in sheet1" back to "Loop Over Items":**  
    - To process all jobs in batches.

19. **Add Gmail node ("Send a message"):**  
    - Configure Gmail OAuth2 credentials.  
    - Send email to your address notifying job search completion and results availability.  
    - Connect output from "Loop Over Items" node after batch processing.

20. **Add Sticky Notes as needed:**  
    - Document credential setup steps for Google Drive, Google Sheets, Google Gemini API, Gmail.  
    - Include usage instructions and links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| Workflow takes about 1 hour to run due to batch processing and waiting to avoid rate limits.                                                                    | Sticky Note: Job search ultimate workflow @jugaldb      |
| Google Cloud credentials are needed for Google Drive, Google Sheets, and Gmail nodes; ensure OAuth2 setup per node.                                              | Sticky Notes 1, 2, 3                                    |
| Google Gemini (PaLM) API Free Tier key required for AI analysis nodes; create project and API key in Google Cloud Console.                                      | Sticky Notes 4, 5                                       |
| LinkedIn job search URL uses filters for last 24 hours, keywords, location, experience level, remote/hybrid/onsite, and easy apply option.                      | Code node "LinkedIn Search URL"                          |
| Job scoring schema includes skills, experience, responsibilities, education, domain fit, and logistics with detailed evidence and gap analysis in JSON output.   | AI Agent node prompt                                     |
| Resume improvement suggestions output is a concise, actionable list tagged by edit type for clarity and prioritization.                                          | AI Agent1 node prompt                                    |
| Email notification informs the user that job search results and resume recommendations are ready in the Google Sheet.                                           | Gmail node "Send a message"                              |
| For best results, keep your resume and Google Sheet filter criteria updated regularly and monitor API usage quotas for Google Gemini and Google services.       | General operational advice                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It adheres strictly to content policies and contains no illegal or offensive material. All manipulated data is legal and public.