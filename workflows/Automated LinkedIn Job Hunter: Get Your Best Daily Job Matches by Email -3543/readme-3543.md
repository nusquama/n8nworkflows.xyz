Automated LinkedIn Job Hunter: Get Your Best Daily Job Matches by Email 

https://n8nworkflows.xyz/workflows/automated-linkedin-job-hunter--get-your-best-daily-job-matches-by-email--3543


# Automated LinkedIn Job Hunter: Get Your Best Daily Job Matches by Email 

### 1. Workflow Overview

This workflow automates the daily retrieval and personalized ranking of LinkedIn job listings, delivering the top 5 matches via email. It targets active job seekers who want to save time by automating job discovery, filtering, and notification based on their resume and preferences.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Input Setup:** Initiates the workflow on a schedule and sets user-specific inputs such as resume file ID, LinkedIn search URL, job preferences, and email recipient.
- **1.2 Resume Download & Text Extraction:** Downloads the user’s resume PDF from Google Drive and extracts its textual content for AI analysis.
- **1.3 LinkedIn Job Scraping:** Uses Apify via HTTP request to scrape recent LinkedIn job listings based on the user’s custom search URL.
- **1.4 Job Data Aggregation & Preparation:** Aggregates scraped job data and combines it into a single text string for AI processing.
- **1.5 AI-Powered Job Matching:** Employs Google Gemini AI agent to analyze and rank jobs against the resume and preferences, producing a structured output.
- **1.6 Output Parsing & Auto-fixing:** Parses and auto-corrects the AI output to ensure structured data integrity.
- **1.7 Email Generation & Sending:** Formats the top 5 job matches into a personalized HTML email and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Input Setup

- **Overview:** This block triggers the workflow on a defined schedule (default daily 8 AM) and sets all user inputs required downstream.
- **Nodes Involved:** `Schedule Trigger`, `Input`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Starts the workflow automatically at configured intervals.
    - Configuration: Default schedule (daily 8 AM); can be customized.
    - Inputs: None (trigger node).
    - Outputs: Triggers `Input` node.
    - Edge Cases: Misconfigured schedule may cause no runs or unexpected runs.

  - **Input**
    - Type: Set
    - Role: Defines static user inputs such as Apify API key, Google Drive Resume File ID, LinkedIn search URL, job preferences, and recipient email.
    - Configuration: Uses key-value pairs to store inputs.
    - Key Expressions: Assigns values like `ApifyAPIKey`, `FileID`, `LinkedInSearchURL`, `Preference`, `EmailAddressToReceiveJobRecommendations`.
    - Inputs: Triggered by `Schedule Trigger`.
    - Outputs: Passes data to `DownloadResume`.
    - Edge Cases: Missing or incorrect input values (e.g., invalid File ID or API key) will cause downstream failures.

#### 2.2 Resume Download & Text Extraction

- **Overview:** Downloads the resume PDF from Google Drive and extracts its text content for AI analysis.
- **Nodes Involved:** `DownloadResume`, `Extract Information from Resume PDF`, `SetResumeField`
- **Node Details:**

  - **DownloadResume**
    - Type: Google Drive
    - Role: Downloads the resume PDF file using the File ID.
    - Configuration: Uses Google Drive credentials; File ID set from `Input` node.
    - Inputs: Receives inputs from `Input`.
    - Outputs: Passes file binary data to `Extract Information from Resume PDF`.
    - Edge Cases: Invalid File ID, permission errors, or file not found.

  - **Extract Information from Resume PDF**
    - Type: Extract From File
    - Role: Extracts text content from the downloaded PDF file.
    - Configuration: Default PDF text extraction.
    - Inputs: Receives binary file from `DownloadResume`.
    - Outputs: Passes extracted text to `SetResumeField`.
    - Edge Cases: Corrupted PDF, extraction failures.

  - **SetResumeField**
    - Type: Set
    - Role: Prepares and sets the extracted resume text into a field for later AI processing.
    - Configuration: Sets a field (e.g., `ResumeText`) with extracted content.
    - Inputs: Receives extracted text.
    - Outputs: Passes data to `ScrapeLinkedin`.
    - Edge Cases: Empty or incomplete extraction.

#### 2.3 LinkedIn Job Scraping

- **Overview:** Scrapes recent LinkedIn job listings using Apify API based on the user’s LinkedIn search URL.
- **Nodes Involved:** `SetResumeField`, `ScrapeLinkedin`, `Aggregate all returned items`
- **Node Details:**

  - **ScrapeLinkedin**
    - Type: HTTP Request
    - Role: Calls Apify API to scrape LinkedIn jobs.
    - Configuration:
      - Method: POST
      - URL: Apify endpoint for LinkedIn job scraping
      - Headers: Includes Apify API key from `Input`
      - Body: JSON containing the LinkedIn search URL and count (e.g., 100 jobs)
    - Inputs: Receives data from `SetResumeField`.
    - Outputs: Passes scraped job data to `Aggregate all returned items`.
    - Edge Cases: API key invalid, rate limits, LinkedIn URL invalid or blocked, network timeouts.

  - **Aggregate all returned items**
    - Type: Aggregate
    - Role: Combines multiple paginated or chunked job listings into a single collection.
    - Configuration: Aggregates all items from the HTTP response.
    - Inputs: Receives scraped job data.
    - Outputs: Passes aggregated jobs to `Combine all job information into a single text string`.
    - Edge Cases: Empty or partial data, aggregation errors.

#### 2.4 Job Data Aggregation & Preparation

- **Overview:** Combines all scraped job information into a single text string for AI input.
- **Nodes Involved:** `Aggregate all returned items`, `Combine all job information into a single text string`
- **Node Details:**

  - **Combine all job information into a single text string**
    - Type: Set
    - Role: Converts aggregated job data into a formatted text string suitable for AI analysis.
    - Configuration: Uses expressions to concatenate job fields (e.g., title, company, description).
    - Inputs: Receives aggregated job data.
    - Outputs: Passes combined text to `AI Agent: Find Best-matched jobs`.
    - Edge Cases: Large data size causing performance issues.

#### 2.5 AI-Powered Job Matching

- **Overview:** Uses Google Gemini AI agent to analyze the resume and job listings, ranking and selecting the top 5 matches.
- **Nodes Involved:** `AI Agent: Find Best-matched jobs`, `Google Gemini Chat Model`, `Google Gemini Chat Model1`, `Structured Output Parser`, `Auto-fixing Output Parser`
- **Node Details:**

  - **Google Gemini Chat Model**
    - Type: Language Model (Google Gemini)
    - Role: Provides AI language model capabilities to the AI Agent.
    - Configuration: Uses Google Gemini credentials.
    - Inputs: Connected as AI language model for `AI Agent`.
    - Outputs: AI-generated responses.
    - Edge Cases: API quota limits, authentication errors.

  - **Google Gemini Chat Model1**
    - Type: Language Model (Google Gemini)
    - Role: Used by the `Auto-fixing Output Parser` to refine AI output.
    - Configuration: Same as above.
    - Inputs: Connected to `Auto-fixing Output Parser`.
    - Outputs: Refined AI output.
    - Edge Cases: Same as above.

  - **AI Agent: Find Best-matched jobs**
    - Type: LangChain AI Agent
    - Role: Core AI logic that compares resume text and job listings, ranks jobs, and outputs structured data.
    - Configuration: Uses prompts incorporating resume text, job data, and user preferences.
    - Inputs: Receives combined job text and resume data.
    - Outputs: Structured job match data to email node.
    - Retry: Enabled on failure.
    - Edge Cases: AI output format errors, timeout, unexpected responses.

  - **Structured Output Parser**
    - Type: LangChain Structured Output Parser
    - Role: Parses AI output into a strict structured format.
    - Configuration: Uses a schema defining expected fields (e.g., job title, company, match reason).
    - Inputs: Receives AI agent output.
    - Outputs: Passes parsed data to `Auto-fixing Output Parser`.
    - Edge Cases: Parsing failures if AI output deviates.

  - **Auto-fixing Output Parser**
    - Type: LangChain Auto-fixing Output Parser
    - Role: Automatically corrects minor AI output errors to conform to the schema.
    - Configuration: Uses Google Gemini Chat Model1 for corrections.
    - Inputs: Receives structured output parser data.
    - Outputs: Passes corrected structured data to `AI Agent`.
    - Edge Cases: Failure to fix major errors.

#### 2.6 Email Generation & Sending

- **Overview:** Formats the top 5 job matches into a personalized HTML email and sends it to the user.
- **Nodes Involved:** `AI Agent: Find Best-matched jobs`, `Email the top job recommendations`
- **Node Details:**

  - **Email the top job recommendations**
    - Type: Gmail
    - Role: Sends the final email with job recommendations.
    - Configuration:
      - Uses Gmail OAuth2 credentials.
      - Email recipient set from `Input` node.
      - Email body is HTML formatted, looping through the top 5 jobs with details and personalized match reasons.
    - Inputs: Receives structured job match data from AI Agent.
    - Outputs: None (terminal node).
    - Edge Cases: Authentication errors, email sending failures, invalid recipient address.

---

### 3. Summary Table

| Node Name                        | Node Type                      | Functional Role                          | Input Node(s)                  | Output Node(s)                       | Sticky Note                         |
|---------------------------------|--------------------------------|----------------------------------------|--------------------------------|------------------------------------|-----------------------------------|
| Schedule Trigger                | Schedule Trigger               | Initiates workflow on schedule         | None                           | Input                              |                                   |
| Input                          | Set                           | Sets user inputs (API keys, URLs, etc.)| Schedule Trigger               | DownloadResume                     |                                   |
| DownloadResume                 | Google Drive                  | Downloads resume PDF                    | Input                          | Extract Information from Resume PDF|                                   |
| Extract Information from Resume PDF | Extract From File             | Extracts text from PDF                  | DownloadResume                 | SetResumeField                    |                                   |
| SetResumeField                 | Set                           | Prepares resume text field              | Extract Information from Resume PDF | ScrapeLinkedin                 |                                   |
| ScrapeLinkedin                 | HTTP Request                  | Scrapes LinkedIn jobs via Apify API    | SetResumeField                 | Aggregate all returned items       |                                   |
| Aggregate all returned items   | Aggregate                     | Aggregates scraped job listings         | ScrapeLinkedin                 | Combine all job information into a single text string |                                   |
| Combine all job information into a single text string | Set                           | Combines job data into text for AI     | Aggregate all returned items   | AI Agent: Find Best-matched jobs   |                                   |
| Google Gemini Chat Model       | Language Model (Google Gemini) | Provides AI language model              | Connected internally to AI Agent | AI Agent: Find Best-matched jobs  |                                   |
| Google Gemini Chat Model1      | Language Model (Google Gemini) | Used for auto-fixing AI output          | Connected internally to Auto-fixing Output Parser | Auto-fixing Output Parser |                                   |
| Structured Output Parser       | LangChain Structured Output Parser | Parses AI output into structured data | AI Agent: Find Best-matched jobs | Auto-fixing Output Parser          |                                   |
| Auto-fixing Output Parser      | LangChain Auto-fixing Output Parser | Auto-corrects AI output errors         | Structured Output Parser       | AI Agent: Find Best-matched jobs   |                                   |
| AI Agent: Find Best-matched jobs | LangChain AI Agent            | Core AI logic for job matching          | Combine all job information into a single text string, Auto-fixing Output Parser | Email the top job recommendations |                                   |
| Email the top job recommendations | Gmail                        | Sends personalized job match email      | AI Agent: Find Best-matched jobs | None                             |                                   |
| Sticky Note                    | Sticky Note                   | Visual notes (empty content in this workflow) | None                        | None                              |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Configure to run daily at 8 AM (or desired frequency/time).

2. **Create an Input node (Set):**
   - Connect from Schedule Trigger.
   - Add key-value pairs for:
     - `ApifyAPIKey`: Your Apify API key.
     - `FileID`: Google Drive File ID of your resume PDF.
     - `LinkedInSearchURL`: Your LinkedIn job search URL (public/incognito).
     - `Preference`: Text describing your job preferences.
     - `EmailAddressToReceiveJobRecommendations`: Your email address.

3. **Create a Google Drive node (DownloadResume):**
   - Connect from Input.
   - Set operation to download file by ID.
   - Use credentials for your Google Drive.
   - Use `FileID` from Input node as file identifier.

4. **Create an Extract From File node (Extract Information from Resume PDF):**
   - Connect from DownloadResume.
   - Configure to extract text from PDF binary data.

5. **Create a Set node (SetResumeField):**
   - Connect from Extract Information from Resume PDF.
   - Set a field (e.g., `ResumeText`) with the extracted text content.

6. **Create an HTTP Request node (ScrapeLinkedin):**
   - Connect from SetResumeField.
   - Configure as POST request to Apify LinkedIn scraping API endpoint.
   - Set headers to include `Authorization: Bearer {{ $json["ApifyAPIKey"] }}`.
   - Set JSON body with:
     - `urls`: Array containing `LinkedInSearchURL`.
     - `count`: Number of jobs to scrape (e.g., 100).
   - Use credentials if required.

7. **Create an Aggregate node (Aggregate all returned items):**
   - Connect from ScrapeLinkedin.
   - Configure to aggregate all job items from the response.

8. **Create a Set node (Combine all job information into a single text string):**
   - Connect from Aggregate all returned items.
   - Use expressions to concatenate job details into a single string for AI input.

9. **Create a LangChain Google Gemini Chat Model node (Google Gemini Chat Model):**
   - Configure with your Google Gemini credentials.
   - No direct input/output connections; used internally by AI Agent.

10. **Create a LangChain Google Gemini Chat Model node (Google Gemini Chat Model1):**
    - Same as above, used for auto-fixing output parser.

11. **Create a LangChain Structured Output Parser node (Structured Output Parser):**
    - Connect internally to AI Agent.
    - Configure schema for expected AI output (job title, company, match reason, etc.).

12. **Create a LangChain Auto-fixing Output Parser node (Auto-fixing Output Parser):**
    - Connect from Structured Output Parser.
    - Configure to use Google Gemini Chat Model1 for corrections.

13. **Create a LangChain AI Agent node (AI Agent: Find Best-matched jobs):**
    - Connect input from Combine all job information into a single text string.
    - Configure to use Google Gemini Chat Model as language model.
    - Set retry on failure enabled.
    - Connect output to Email node.
    - Configure prompt to analyze resume text, job listings, and preferences, and output top 5 matches in structured format.

14. **Create a Gmail node (Email the top job recommendations):**
    - Connect from AI Agent node.
    - Use Gmail OAuth2 credentials for sending.
    - Set recipient email from `EmailAddressToReceiveJobRecommendations`.
    - Compose HTML email body looping through top 5 jobs with company, title, industry, match reason, and application link.

15. **Connect all nodes as per above steps.**

16. **Test the workflow end-to-end, verifying:**
    - Resume downloads and extracts correctly.
    - LinkedIn scraping returns job data.
    - AI agent ranks and outputs structured matches.
    - Email sends successfully with correct content.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Video walkthrough available showing setup and configuration steps.                                            | Refer to the original template documentation or video link provided with the workflow template. |
| Apify account required for LinkedIn scraping; sign up at [https://apify.com/](https://apify.com/)              | Needed to obtain API key for scraping LinkedIn job listings.                                     |
| Google Drive and Gmail credentials must be configured in n8n credentials section.                              | Required for resume download and email sending respectively.                                     |
| Google Gemini AI credentials and access required for AI Agent nodes.                                          | Used for AI-powered job matching and output parsing.                                            |
| LinkedIn search URL must be public and obtained via incognito/private browser to avoid login/session issues.  | Ensures scraping works without authentication.                                                  |
| Cost estimate: approximately $0.1 USD per day based on API usage and AI calls.                                 | Budget consideration for running the workflow daily.                                            |
| Customize AI prompt and output schema to adjust number of job matches or email formatting as needed.          | Advanced customization for personalized experience.                                            |

---

This documentation provides a complete, structured understanding of the "Automated LinkedIn Job Hunter" workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.