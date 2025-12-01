Automate Job Applications with AI Resume Tailoring using GPT-4o, LinkedIn & Gmail

https://n8nworkflows.xyz/workflows/automate-job-applications-with-ai-resume-tailoring-using-gpt-4o--linkedin---gmail-11215


# Automate Job Applications with AI Resume Tailoring using GPT-4o, LinkedIn & Gmail

### 1. Workflow Overview

This workflow automates the end-to-end process of applying for jobs by leveraging AI-driven resume tailoring, real-time job scraping, and personalized email outreach. It is designed for job seekers aiming to efficiently find relevant positions, customize their resumes for each role, and contact hiring managers with tailored email drafts.

The workflow is logically divided into the following blocks:

- **1.1 Input & Data Acquisition**: Trigger the workflow manually, fetch the master resume from Google Docs, and scrape LinkedIn job postings using Apify.
- **1.2 AI Filtering & Job Matching**: Use GPT-4o-mini to evaluate job descriptions against the candidate’s skills, then filter and limit to top relevant jobs.
- **1.3 Resume Tailoring & Formatting**: Use GPT-4o to rewrite the resume based on each selected job description and convert the output to HTML.
- **1.4 Document Creation & Sharing**: Create a new Google Doc for each tailored resume, upload the HTML content, and set sharing permissions.
- **1.5 Contact Enrichment & Email Drafting**: Retrieve hiring manager emails via Anymail Finder, validate them, and create personalized Gmail drafts for outreach.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Data Acquisition

- **Overview:**  
  This block initializes the workflow manually, retrieves the candidate’s master resume from Google Docs, and scrapes LinkedIn job postings through Apify for further AI processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get a document (Google Docs)  
  - HTTP Request (Apify LinkedIn scraping)  

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point for the workflow, activated manually by the user.  
     - Inputs: None  
     - Outputs: Triggers the next node to fetch the resume.  
     - Failures: None expected unless manual trigger fails to activate.  

  2. **Get a document**  
     - Type: Google Docs node  
     - Role: Fetch the master resume content from Google Docs by specifying the document URL/ID.  
     - Configuration: Operation set to "get" with a placeholder for the Google Doc ID (`[PASTE_YOUR_GOOGLE_DOC_ID_HERE]`).  
     - Inputs: Trigger from manual node  
     - Outputs: JSON containing the document content field.  
     - Failures: Invalid or missing document ID, permission/authentication errors.  
     - Requires: Google Docs OAuth2 credentials.  

  3. **HTTP Request (Apify LinkedIn scraping)**  
     - Type: HTTP Request node  
     - Role: Posts a JSON payload to Apify API to scrape LinkedIn job postings based on a specified search URL and keywords.  
     - Configuration:  
       - URL: Apify endpoint for running a LinkedIn job scraper act synchronously.  
       - Method: POST  
       - Headers: Accept JSON, Authorization Bearer token `[YOUR_APIFY_API_KEY]`.  
       - Body: JSON specifying count=100 jobs, scrapeCompany=true, and LinkedIn job search URL with keywords like "AI Automation".  
     - Inputs: Output from "Get a document" (to sequence the workflow)  
     - Outputs: Job data from LinkedIn in JSON format.  
     - Failures: API key invalid/expired, network issues, LinkedIn blocking scraping.  
     - Requires: Valid Apify API key.

---

#### 2.2 AI Filtering & Job Matching

- **Overview:**  
  This block uses GPT-4o-mini to analyze scraped job descriptions and decide which jobs fit the candidate’s profile. It then filters for positive matches and limits the number of jobs to process.

- **Nodes Involved:**  
  - Limit (max 10 jobs)  
  - Message a model (GPT-4o-mini AI screening)  
  - Filter (filter jobs with verdict true and company website present)  

- **Node Details:**

  1. **Limit**  
     - Type: Limit node  
     - Role: Restricts the number of job items to 10 for manageable processing.  
     - Inputs: Jobs from Apify scraping  
     - Outputs: Limited job list  
     - Failures: None typical, unless input is empty.  

  2. **Message a model (GPT-4o-mini)**  
     - Type: LangChain OpenAI node  
     - Role: Sends each job description to GPT-4.1-mini model to evaluate the candidate’s fit.  
     - Configuration:  
       - Model: GPT-4.1-mini  
       - Temperature: 0.7 (balanced creativity and precision)  
       - Message prompt: System message as job filtering assistant; user message includes candidate’s skills/summary placeholder and job description JSON.  
       - Output: JSON with `verdict` key as string "true" or "false".  
     - Inputs: Limited jobs  
     - Outputs: Each job with AI verdict  
     - Failures: API key issues, prompt formatting errors, output parsing failures.  

  3. **Filter**  
     - Type: Filter node  
     - Role: Keeps only jobs where AI verdict is "true" and company website is non-empty.  
     - Conditions:  
       - verdict equals "true" (string)  
       - companyWebsite field not empty  
     - Inputs: AI model output  
     - Outputs: Filtered relevant jobs  
     - Failures: Missing fields in input JSON, expression evaluation failures.

---

#### 2.3 Resume Tailoring & Formatting

- **Overview:**  
  GPT-4o rewrites the candidate’s resume customized for each relevant job, then converts the resume into HTML for document creation.

- **Nodes Involved:**  
  - Limit1 (max 4 jobs for resume tailoring)  
  - Message a model1 (GPT-4o resume rewriting)  
  - Markdown (Markdown to HTML conversion)  

- **Node Details:**

  1. **Limit1**  
     - Type: Limit node  
     - Role: Caps the number of jobs processed for resume tailoring to 4 to avoid overload.  
     - Inputs: Filtered jobs from previous block  
     - Outputs: Limited job list for resume rewriting  
     - Failures: None typical.  

  2. **Message a model1 (GPT-4o)**  
     - Type: LangChain OpenAI node  
     - Role: Sends job description and master resume content to GPT-4.1 for rewriting the resume tailored to the job.  
     - Configuration:  
       - Model: GPT-4.1  
       - Temperature: 0.7  
       - Prompt includes candidate info placeholder, job description JSON, and original resume text from Google Docs.  
       - Output: Tailored resume in Markdown format, plain text (no code blocks).  
     - Inputs: Limited jobs and master resume content  
     - Outputs: Markdown text of tailored resume  
     - Failures: API key issues, prompt formatting, large input size.  

  3. **Markdown**  
     - Type: Markdown node  
     - Role: Converts Markdown resume output into HTML for uploading to Google Docs.  
     - Configuration: Mode set to "markdownToHtml"  
     - Inputs: Tailored resume Markdown from GPT  
     - Outputs: HTML content  
     - Failures: Invalid Markdown syntax, conversion errors.

---

#### 2.4 Document Creation & Sharing

- **Overview:**  
  This block creates a Google Doc for each tailored resume, uploads the HTML content, and configures sharing permissions for easy access.

- **Nodes Involved:**  
  - Create a document (Google Docs)  
  - Share file (Google Drive)  
  - HTTP Request1 (Google Docs API PATCH upload)  

- **Node Details:**

  1. **Create a document**  
     - Type: Google Docs node  
     - Role: Creates a new Google Doc titled "Tailored Resume - [Candidate Name]" for each customized resume.  
     - Configuration: Title set, folder left as default or configurable.  
     - Inputs: HTML resume content from Markdown node  
     - Outputs: Document metadata including new document ID  
     - Failures: Authentication errors, API limits.  

  2. **Share file**  
     - Type: Google Drive node  
     - Role: Updates sharing permissions of the created Google Doc to "writer" role accessible by "anyone".  
     - Configuration: Permissions set for anyone with link as writer.  
     - Inputs: Document ID from "Create a document"  
     - Outputs: Shared file metadata  
     - Failures: Permission denied, incorrect file ID.  

  3. **HTTP Request1**  
     - Type: HTTP Request node  
     - Role: Uploads the converted HTML content to the Google Doc using Drive API PATCH method with content type "text/html".  
     - Configuration:  
       - URL dynamically constructed with the created document ID  
       - Method: PATCH  
       - Content-Type header: text/html  
       - Body: HTML content from Markdown node  
       - Authentication: Google Docs OAuth2 credentials  
     - Inputs: Document ID & HTML resume  
     - Outputs: Confirmation of upload  
     - Failures: Upload errors, quota exceeded, invalid auth.

---

#### 2.5 Contact Enrichment & Email Drafting

- **Overview:**  
  This block enriches company data by finding CEO/hiring manager emails, validates them, and drafts personalized outreach emails in Gmail with resume links.

- **Nodes Involved:**  
  - HTTP Request2 (Anymail Finder API)  
  - Filter1 (email validation)  
  - Create a draft (Gmail draft creation)  

- **Node Details:**

  1. **HTTP Request2**  
     - Type: HTTP Request node  
     - Role: Posts to Anymail Finder API to get decision-maker (CEO) email based on company domain.  
     - Configuration:  
       - URL: Anymail Finder API endpoint for decision-maker emails  
       - Method: POST  
       - Body parameters: domain from job’s companyWebsite, decision_maker_category set to "ceo"  
       - Header: Authorization with `[YOUR_ANYMAIL_FINDER_API_KEY]`  
     - Inputs: Company website from filtered jobs  
     - Outputs: Email data JSON  
     - Failures: API key invalid, domain missing, API rate limits.  

  2. **Filter1**  
     - Type: Filter node  
     - Role: Filters out entries where email field is empty to avoid sending drafts to invalid addresses.  
     - Condition: email field not empty  
     - Inputs: Email data from Anymail Finder  
     - Outputs: Valid emails only  
     - Failures: Missing email field, JSON structure unexpected.  

  3. **Create a draft**  
     - Type: Gmail node  
     - Role: Creates a Gmail draft email addressed to the decision-maker with a personalized message and a link to the tailored resume Google Doc.  
     - Configuration:  
       - Subject: "Re : [Job Title]" dynamically from job data  
       - Message: Personalized pitch referencing AI tailoring, job/company name, and link to resume doc  
       - Resource: Draft message  
     - Inputs: Valid email addresses  
     - Outputs: Gmail draft metadata  
     - Failures: Authentication failures, invalid recipient email, SMTP errors.  
     - Requires: Gmail OAuth2 credentials.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                            |
|-------------------------------|--------------------------------|----------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Workflow entry point                    | None                             | Get a document                    |                                                                                                                        |
| Get a document                | Google Docs                    | Fetch master resume content             | When clicking ‘Execute workflow’ | HTTP Request                     | STICKY 1 – Input & Scraping: Fetches your master resume, scrapes LinkedIn job postings, and prepares raw data for AI filtering. |
| HTTP Request                 | HTTP Request                   | Scrape LinkedIn jobs via Apify          | Get a document                   | Limit                            | STICKY 1 – Input & Scraping                                                                                             |
| Limit                       | Limit                         | Limit jobs to 10                        | HTTP Request                    | Message a model                  | STICKY 2 – AI Filtering: GPT analyzes job descriptions, removes irrelevant leads, limits top matches.                   |
| Message a model              | LangChain OpenAI (GPT-4o-mini) | AI job relevance screening              | Limit                          | Filter                          | STICKY 2 – AI Filtering                                                                                                |
| Filter                      | Filter                        | Filter jobs with fit verdict and website | Message a model                 | Limit1                          | STICKY 2 – AI Filtering                                                                                                |
| Limit1                      | Limit                        | Limit jobs to 4 for resume tailoring    | Filter                         | Message a model1                | STICKY 3 – Resume Writer: GPT rewrites resume, converts output to HTML.                                                |
| Message a model1             | LangChain OpenAI (GPT-4o)     | Tailor resume per job                    | Limit1                        | Markdown                       | STICKY 3 – Resume Writer                                                                                               |
| Markdown                    | Markdown                      | Convert Markdown resume to HTML          | Message a model1               | Create a document              | STICKY 3 – Resume Writer                                                                                               |
| Create a document            | Google Docs                   | Create new Google Doc for tailored resume | Markdown                    | Share file                    | STICKY 4 – Google Docs: Creates doc, uploads resume, adjusts sharing.                                                 |
| Share file                  | Google Drive                  | Share Google Doc publicly with write access | Create a document            | HTTP Request1                 | STICKY 4 – Google Docs                                                                                                |
| HTTP Request1               | HTTP Request                 | Upload HTML resume content to Google Doc | Share file                   | HTTP Request2                 | STICKY 4 – Google Docs                                                                                                |
| HTTP Request2               | HTTP Request                 | Query Anymail Finder for CEO email       | HTTP Request1                | Filter1                      | STICKY 5 – Email Outreach: Find CEO email, validate, create Gmail draft.                                              |
| Filter1                    | Filter                      | Validate presence of email address        | HTTP Request2               | Create a draft               | STICKY 5 – Email Outreach                                                                                            |
| Create a draft             | Gmail                        | Create personalized email draft           | Filter1                     | None                        | STICKY 5 – Email Outreach                                                                                            |
| Main Sticky Note           | Sticky Note                  | Overview and instructions                 | None                        | None                        | See Section 5                                                                                                          |
| STICKY 1 – Input & Scraping | Sticky Note                  | Describes input and scraping block        | None                        | None                        | See above                                                                                                             |
| STICKY 2 – AI Filtering     | Sticky Note                  | Describes AI filtering block               | None                        | None                        | See above                                                                                                             |
| STICKY 3 – Resume Writer    | Sticky Note                  | Describes resume tailoring block          | None                        | None                        | See above                                                                                                             |
| STICKY 4 – Google Docs      | Sticky Note                  | Describes Google Docs creation & sharing  | None                        | None                        | See above                                                                                                             |
| STICKY 5 – Email Outreach   | Sticky Note                  | Describes contact enrichment & outreach   | None                        | None                        | See above                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Manual start of workflow.

2. **Add Google Docs node:**  
   - Name: "Get a document"  
   - Operation: Get document content  
   - DocumentURL: Paste your Google Doc ID with your master resume.  
   - Connect output of manual trigger to this node.  
   - Set up Google Docs OAuth2 credentials.

3. **Add HTTP Request node:**  
   - Name: "HTTP Request"  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/hKByXkMQaC5Qt9UMN/run-sync-get-dataset-items`  
   - Headers: Accept: application/json, Authorization: Bearer `[YOUR_APIFY_API_KEY]`  
   - Body (JSON):  
     ```json
     {
       "count": 100,
       "scrapeCompany": true,
       "urls": [
         "https://www.linkedin.com/jobs/search/?currentJobId=4250801865&geoId=103644278&keywords=ai%20automation&origin=JOB_COLLECTION_PAGE_LOCATION_AUTOCOMPLETE&refresh=true"
       ]
     }
     ```  
   - Connect output of "Get a document" node here.

4. **Add Limit node:**  
   - Name: "Limit"  
   - Max Items: 10  
   - Connect output of HTTP Request node here.

5. **Add LangChain OpenAI node:**  
   - Name: "Message a model"  
   - Model: GPT-4.1-mini  
   - Temperature: 0.7  
   - Messages:  
     - System: `"You're a helpful, intelligent job filtering assistant."`  
     - User: Custom prompt including your resume summary, skills, and job description from input JSON. (Use expression to insert current job JSON)  
   - Enable JSON output parsing.  
   - Connect output of "Limit" node here.  
   - Configure OpenAI credentials.

6. **Add Filter node:**  
   - Name: "Filter"  
   - Conditions:  
     - `{{$json.message.content.verdict}}` equals `"true"`  
     - `{{$json.companyWebsite}}` is not empty  
   - Connect output of "Message a model" node here.

7. **Add second Limit node:**  
   - Name: "Limit1"  
   - Max Items: 4  
   - Connect output of "Filter" node here.

8. **Add second LangChain OpenAI node:**  
   - Name: "Message a model1"  
   - Model: GPT-4.1  
   - Temperature: 0.7  
   - Messages:  
     - System: `"You're a helpful, intelligent resume customization assistant."`  
     - User: Custom prompt containing your resume summary, skills, job description JSON from current item, and the original resume content fetched in step 2.  
     - Specify output format as Markdown without code block ticks.  
   - Connect output of "Limit1" node here.

9. **Add Markdown node:**  
   - Name: "Markdown"  
   - Mode: markdownToHtml  
   - Input: Use expression to get Markdown text from "Message a model1" output.  
   - Connect output of "Message a model1" node here.

10. **Add Google Docs node:**  
    - Name: "Create a document"  
    - Operation: Create  
    - Title: "Tailored Resume - [Candidate Name]" (replace placeholder or make dynamic)  
    - FolderId: Optional, use default or specific drive folder  
    - Connect output of "Markdown" node here.

11. **Add Google Drive node:**  
    - Name: "Share file"  
    - Operation: Share  
    - FileId: Use expression to get document ID from "Create a document" node output  
    - Permissions: Role "writer", Type "anyone"  
    - Connect output of "Create a document" node here.

12. **Add HTTP Request node:**  
    - Name: "HTTP Request1"  
    - Method: PATCH  
    - URL:  
      `https://www.googleapis.com/upload/drive/v3/files/{{ $json.id }}?uploadType=media` (use document ID from "Create a document")  
    - Headers: Content-Type: text/html  
    - Body: HTML content from "Markdown" node  
    - Authentication: Google Docs OAuth2  
    - Connect output of "Share file" node here.

13. **Add HTTP Request node:**  
    - Name: "HTTP Request2"  
    - Method: POST  
    - URL: `https://api.anymailfinder.com/v5.1/find-email/decision-maker`  
    - Headers: Authorization with your Anymail Finder API key  
    - Body parameters:  
      - domain: use companyWebsite field from "Limit" or filtered job item  
      - decision_maker_category: "ceo"  
    - Connect output of "HTTP Request1" node here.

14. **Add Filter node:**  
    - Name: "Filter1"  
    - Condition: email field is not empty  
    - Connect output of "HTTP Request2" node here.

15. **Add Gmail node:**  
    - Name: "Create a draft"  
    - Operation: Create draft  
    - Subject: `"Re : {{ $json.title }}"`  
    - Message: Personalized email referencing AI tailoring, job/company, and link to resume Google Doc (`https://docs.google.com/document/d/{{ documentId }}`)  
    - Connect output of "Filter1" node here.  
    - Configure Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow acts as a personal AI recruiter automating job scraping, AI filtering, resume tailoring, document creation, and cold email outreach.              | Main Sticky Note in the workflow                  |
| Requires API keys and OAuth credentials for Google Docs/Drive, Apify, OpenAI (GPT-4o and GPT-4o-mini), Anymail Finder, and Gmail.                              | Setup Requirements section in workflow notes     |
| LinkedIn job scraping uses Apify; ensure your Apify account is active and API key is valid.                                                                     | Apify API documentation                           |
| GPT prompts include placeholders `[INSERT YOUR RESUME SUMMARY AND SKILLS HERE]` to be customized by the user for best performance.                             | Prompt notes in LangChain OpenAI nodes            |
| Gmail draft includes a message template explaining the AI-driven tailoring process and includes a link to the tailored resume Google Doc for recruiter review. | Gmail draft node configuration                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.