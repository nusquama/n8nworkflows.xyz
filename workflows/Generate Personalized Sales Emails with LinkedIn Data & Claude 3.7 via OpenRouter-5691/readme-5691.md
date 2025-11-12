Generate Personalized Sales Emails with LinkedIn Data & Claude 3.7 via OpenRouter

https://n8nworkflows.xyz/workflows/generate-personalized-sales-emails-with-linkedin-data---claude-3-7-via-openrouter-5691


# Generate Personalized Sales Emails with LinkedIn Data & Claude 3.7 via OpenRouter

### 1. Workflow Overview

This workflow automates the generation and sending of personalized sales emails by leveraging LinkedIn data about leads and their companies, combined with advanced AI text generation (Claude 3.7 via OpenRouter). It targets sales or outreach teams seeking to scale personalized cold email campaigns using enriched LinkedIn profiles without manual research.

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger to start, fetching leads from Google Sheets.
- **1.2 LinkedIn Data Retrieval:** Searching and filtering LinkedIn URLs for persons and companies using Apify Google Search and LinkedIn profile scrapers.
- **1.3 Data Aggregation and Conditional Logic:** Aggregating search results, filtering valid URLs, and conditional branching based on existing company data.
- **1.4 AI Email Generation:** Using LangChain with Claude 3.7 model to generate personalized email content based on collected LinkedIn data and preset company info.
- **1.5 Email Sending:** Sending generated emails through Gmail with dynamic personalization.
- **1.6 Supporting Data Setup:** Static data node to configure company info and sender details for email personalization.
- **1.7 Output Parsing:** Structured output parser to extract subject and body from AI output JSON.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow manually, then retrieves the list of leads from a Google Sheet.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Leads  
- Loop Over Items

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point requiring manual execution.  
  - No parameters.  
  - Output: Connected to "Get Leads".

- **Get Leads**  
  - Type: Google Sheets  
  - Role: Reads leads data (including names, emails, companies) from a specific Google Sheet tab ("Leads / Unprocessed") in a Google Sheets document.  
  - Config: OAuth2 credential for Google Sheets.  
  - Output: List of lead items to process.  
  - Connections: Outputs to "Loop Over Items".

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each lead one by one in batches (default batch size).  
  - Input: From "Get Leads".  
  - Output: Two outputs: main (empty, unused), secondary to "Google Search for Person LinkedIn".  
  - Potential issues: Large lead lists may cause timeouts or rate limits.

---

#### 2.2 LinkedIn Data Retrieval

**Overview:**  
Performs Google searches on LinkedIn for person and company profiles, filters URLs with regex to separate personal profiles from company pages, then fetches detailed profiles via Apify LinkedIn scrapers.

**Nodes Involved:**  
- Google Search for Person LinkedIn  
- Only People Links  
- Aggregate  
- Get LinkedIn Person  
- Current Company Exists?  
- Set Company URL  
- Google Search for Company LinkedIn  
- Only Company Links  
- Aggregate1  
- Set Company URL1  
- Get LinkedIn Company

**Node Details:**

- **Google Search for Person LinkedIn**  
  - Type: HTTP Request  
  - Role: Calls Apify’s rag-web-browser actor to perform a Google search for the lead’s full name + company restricted to site:linkedin.com.  
  - Config: POST request with query parameter `"First Name Last Name Company site:linkedin.com"` and maxResults=10.  
  - Auth: HTTP Header Auth with Apify API Key.  
  - Output: Search results with URLs.

- **Only People Links**  
  - Type: Filter  
  - Role: Filters search results to only personal LinkedIn profile URLs using regex pattern `^https:\/\/www\.linkedin\.com\/in\/[a-zA-Z0-9\-_%]+\/?$`.  
  - Input: From "Google Search for Person LinkedIn".  
  - Output: Filtered person profile URLs.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates filtered personal profile search results into a single JSON array field `results`.  
  - Input: From "Only People Links".  
  - Output: Aggregated data passed to "Get LinkedIn Person".

- **Get LinkedIn Person**  
  - Type: HTTP Request  
  - Role: Calls Apify’s linkedin-profile-detail scraper to get detailed LinkedIn profile data based on the first filtered URL.  
  - Config: POST request with parameter `username` set to the first search result URL.  
  - Auth: HTTP Header Auth with Apify API Key.  
  - On error: Continue workflow without halting.  
  - Output: Detailed person profile JSON.

- **Current Company Exists?**  
  - Type: If  
  - Role: Checks if the person profile contains a current company URL (`basic_info.current_company_url`).  
  - True output: To "Set Company URL".  
  - False output: To "Google Search for Company LinkedIn".

- **Set Company URL**  
  - Type: Set  
  - Role: Sets variable `company_url` to the current company URL from the person profile for fetching company data.  
  - Output: To "Get LinkedIn Company".

- **Google Search for Company LinkedIn**  
  - Type: HTTP Request  
  - Role: Similar to person search but queries only company name with `site:linkedin.com`.  
  - Output: Raw company search results.

- **Only Company Links**  
  - Type: Filter  
  - Role: Filters company URLs using regex pattern `^https:\/\/www\.linkedin\.com\/company\/[a-zA-Z0-9\-_%]+\/?$`.  
  - Output: Filtered company profile URLs.

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates filtered company profile URLs.  
  - Output: To "Set Company URL1".

- **Set Company URL1**  
  - Type: Set  
  - Role: Sets variable `company_url` to first filtered company URL.  
  - Output: To "Get LinkedIn Company".

- **Get LinkedIn Company**  
  - Type: HTTP Request  
  - Role: Calls Apify’s linkedin-company-detail scraper to get detailed company profile data based on the `company_url`.  
  - Auth: HTTP Header Auth with Apify API Key.  
  - Output: Detailed company profile JSON.

**Edge Cases & Failure Types:**  
- Google search results may be empty or inaccurate (e.g., common names).  
- Regex filters may exclude valid profiles or include false positives.  
- Apify API calls may timeout or return errors; node configured to continue.  
- Missing company URL triggers fallback to company search.  
- Rate limiting or API quota exhaustion possible.

---

#### 2.3 Data Setup and Personalization Context

**Overview:**  
Static data node sets company information and email sender details used for AI email generation and sending.

**Nodes Involved:**  
- Set Data

**Node Details:**

- **Set Data**  
  - Type: Set  
  - Role: Defines key personalization fields: company_name, email_address, company_info (long marketing text about AAAutomations), sender_name.  
  - Used downstream in AI prompt and Gmail node.  
  - No inputs, triggered from "Get LinkedIn Company".  
  - Edge Cases: Static data; requires manual updates to reflect the client’s actual company.

---

#### 2.4 AI Email Generation

**Overview:**  
Generates a personalized sales email draft using collected LinkedIn data and the static company info via Claude 3.7 model hosted on OpenRouter, with structured JSON output parsing.

**Nodes Involved:**  
- Generate Personalized Email  
- OpenRouter Chat Model  
- Structured Output Parser

**Node Details:**

- **Generate Personalized Email**  
  - Type: LangChain Chain LLM  
  - Role: Constructs a prompt with JSON stringified LinkedIn person and company data plus static company info.  
  - Prompt instructions specify tone, style, personalization requirements, and output format (JSON with subject and body).  
  - Connected downstream from "Set Data".  
  - Output: Raw AI response JSON.

- **OpenRouter Chat Model**  
  - Type: LangChain LM Chat OpenRouter  
  - Role: Calls Claude 3.7 via OpenRouter API.  
  - Model: `anthropic/claude-3.7-sonnet`  
  - Credential: OpenRouter API key.  
  - Input: From "Generate Personalized Email".  
  - Output: AI-generated email content (JSON).

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI response JSON to extract `subject` and `body`.  
  - Example schema provided to guide parsing.  
  - Output: Clean JSON for email sending.

**Edge Cases:**  
- AI may return malformed or incomplete JSON; parser expects strict format.  
- Model latency or API errors possible.  
- Prompt maintenance critical for quality output.

---

#### 2.5 Email Sending

**Overview:**  
Sends the generated personalized email to the lead’s email address via Gmail API.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - Type: Gmail  
  - Role: Sends email with subject and body from AI output.  
  - Recipient: Lead email from "Get Leads".  
  - Sender name and reply-to set from "Set Data".  
  - Credential: OAuth2 Gmail credentials.  
  - Input: Email subject and body from "Generate Personalized Email".  
  - Output: Email sent confirmation.  
  - Edge Cases: Email sending failures due to auth issues, quota limits, or invalid addresses.

---

#### 2.6 Notes and Documentation

**Nodes Involved:**  
- Sticky Note (3 instances)

**Purpose:**  
- Describe data gathering strategy for person and company LinkedIn profiles using Apify and Google search.  
- Explain email generation and sending logic.  
- Provide user instructions on setting static company data.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                   | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                  |
|-------------------------------|---------------------------------|---------------------------------|---------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Workflow start trigger           |                                 | Get Leads                       |                                                                                              |
| Get Leads                      | Google Sheets                   | Fetch leads data                 | When clicking ‘Execute workflow’ | Loop Over Items                 |                                                                                              |
| Loop Over Items                | SplitInBatches                  | Process each lead individually   | Get Leads                      | Google Search for Person LinkedIn|                                                                                              |
| Google Search for Person LinkedIn | HTTP Request                   | Google search LinkedIn profiles  | Loop Over Items                | Only People Links               |                                                                                              |
| Only People Links              | Filter                         | Filter personal LinkedIn URLs    | Google Search for Person LinkedIn | Aggregate                      |                                                                                              |
| Aggregate                     | Aggregate                      | Aggregate person search results  | Only People Links              | Get LinkedIn Person             |                                                                                              |
| Get LinkedIn Person           | HTTP Request                   | Fetch detailed LinkedIn profile  | Aggregate                     | Current Company Exists?          |                                                                                              |
| Current Company Exists?       | If                             | Check for company URL in profile | Get LinkedIn Person           | Set Company URL (true), Google Search for Company LinkedIn (false) |                                                                                              |
| Set Company URL               | Set                            | Set company URL from profile     | Current Company Exists? (true) | Get LinkedIn Company            |                                                                                              |
| Google Search for Company LinkedIn | HTTP Request                   | Google search LinkedIn companies | Current Company Exists? (false) | Only Company Links             |                                                                                              |
| Only Company Links            | Filter                         | Filter company LinkedIn URLs     | Google Search for Company LinkedIn | Aggregate1                     |                                                                                              |
| Aggregate1                   | Aggregate                      | Aggregate company search results | Only Company Links            | Set Company URL1                |                                                                                              |
| Set Company URL1             | Set                            | Set company URL from search      | Aggregate1                    | Get LinkedIn Company            |                                                                                              |
| Get LinkedIn Company         | HTTP Request                   | Fetch detailed LinkedIn company  | Set Company URL, Set Company URL1 | Set Data                      |                                                                                              |
| Set Data                    | Set                            | Static company and sender info   | Get LinkedIn Company           | Generate Personalized Email     |                                                                                              |
| Generate Personalized Email  | LangChain Chain LLM            | Generate email draft via Claude 3.7 | Set Data                      | Gmail                         |                                                                                              |
| OpenRouter Chat Model       | LangChain LM Chat OpenRouter   | Claude 3.7 AI model call         | Generate Personalized Email    | Structured Output Parser        |                                                                                              |
| Structured Output Parser    | LangChain Structured Output Parser | Parse AI JSON output             | OpenRouter Chat Model          | Generate Personalized Email     |                                                                                              |
| Gmail                      | Gmail                          | Send personalized email          | Generate Personalized Email    | Loop Over Items                |                                                                                              |
| Sticky Note                 | Sticky Note                    | Explain LinkedIn person profile retrieval |                             |                               | ## Get Person Profile data from LinkedIn ... Apify => Google search for "{{ First Name }} {{ Last Name }} {{ Company }} site:linkedin.com" ... |
| Sticky Note1                | Sticky Note                    | Explain LinkedIn company profile retrieval |                             |                               | ## Get Company Profile data from LinkedIn ... Apify => Google search for {{ Company }} site:linkedin.com ...                    |
| Sticky Note2                | Sticky Note                    | Explain email drafting and sending |                             |                               | ## Write + Send the email - SET YOUR DATA HERE - LLM drafts your email - Send from Gmail account                                |
| Sticky Note3                | Sticky Note                    | Instruction to set static data   |                             |                               | ## Set your data - Set data about your company here so that the bot will customize your offer based on your lead.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters.

2. **Add Google Sheets Node**  
   - Name: "Get Leads"  
   - Connect input from the manual trigger.  
   - Configure to read from the specific Google Sheets document and sheet tab ("Leads / Unprocessed").  
   - Use OAuth2 credentials for Google Sheets.

3. **Add SplitInBatches Node**  
   - Name: "Loop Over Items"  
   - Connect input from "Get Leads".  
   - Default batch size (process items one by one).

4. **Add HTTP Request Node for Person Google Search**  
   - Name: "Google Search for Person LinkedIn"  
   - Connect input from the secondary output of "Loop Over Items".  
   - Configure POST request to Apify rag-web-browser actor with body parameter:  
     `{ "query": "{{ $json['First Name'] }} {{ $json['Last Name'] }} {{ $json['Company'] }} site:linkedin.com", "maxResults": 10 }`.  
   - Use HTTP Header Auth with Apify API key credential.  
   - Set "Always Output Data" to true.

5. **Add Filter Node to Select Person URLs**  
   - Name: "Only People Links"  
   - Connect input from "Google Search for Person LinkedIn".  
   - Add regex condition: `^https:\/\/www\.linkedin\.com\/in\/[a-zA-Z0-9\-_%]+\/?$` applied to `searchResult.url`.  
   - Case sensitive, strict validation.

6. **Add Aggregate Node for Person Search Results**  
   - Name: "Aggregate"  
   - Connect input from "Only People Links".  
   - Aggregate all item data on field `searchResult` into `results`.

7. **Add HTTP Request Node for LinkedIn Person Details**  
   - Name: "Get LinkedIn Person"  
   - Connect input from "Aggregate".  
   - Configure POST request to Apify linkedin-profile-detail scraper with parameter `username` from first item in aggregated results.  
   - Use HTTP Header Auth with Apify API key.  
   - Set "On Error" to "Continue Regular Output".

8. **Add If Node to Check Current Company URL**  
   - Name: "Current Company Exists?"  
   - Connect input from "Get LinkedIn Person".  
   - Condition: Check if `basic_info.current_company_url` exists and is not empty.

9. **Add Set Node to Set Company URL from Person Profile**  
   - Name: "Set Company URL"  
   - Connect from "Current Company Exists?" true branch.  
   - Set JSON output: `{ "company_url": "{{ $json.basic_info.current_company_url }}" }`.

10. **Add HTTP Request Node to Search Company LinkedIn**  
    - Name: "Google Search for Company LinkedIn"  
    - Connect from "Current Company Exists?" false branch.  
    - Configure POST request to Apify rag-web-browser actor with body parameter:  
      `{ "query": "{{ $json['Company'] }} site:linkedin.com" }`.  
    - Use HTTP Header Auth with Apify API key.  
    - Set "Always Output Data" to true.

11. **Add Filter Node to Select Company URLs**  
    - Name: "Only Company Links"  
    - Connect input from "Google Search for Company LinkedIn".  
    - Regex condition: `^https:\/\/www\.linkedin\.com\/company\/[a-zA-Z0-9\-_%]+\/?$` on `searchResult.url`.

12. **Add Aggregate Node for Company Search Results**  
    - Name: "Aggregate1"  
    - Connect input from "Only Company Links".  
    - Aggregate all item data on `searchResult` into `results`.

13. **Add Set Node to Set Company URL from Company Search**  
    - Name: "Set Company URL1"  
    - Connect input from "Aggregate1".  
    - Set JSON output: `{ "company_url": "{{ $json.results.first().searchResult.url }}" }`.

14. **Add HTTP Request Node for LinkedIn Company Details**  
    - Name: "Get LinkedIn Company"  
    - Connect inputs from both "Set Company URL" and "Set Company URL1".  
    - Configure POST request to Apify linkedin-company-detail scraper with parameter `identifier` set from `company_url`.  
    - Use HTTP Header Auth with Apify API key.  
    - Set "Always Output Data" to true.

15. **Add Set Node for Static Company and Sender Data**  
    - Name: "Set Data"  
    - Connect input from "Get LinkedIn Company".  
    - Set multiple fields:  
      - `company_name` (string)  
      - `email_address` (string)  
      - `company_info` (large marketing description string)  
      - `sender_name` (string)  
    - Populate with your own company data for personalization.

16. **Add LangChain Chain LLM Node for Email Generation**  
    - Name: "Generate Personalized Email"  
    - Connect input from "Set Data".  
    - Configure prompt template embedding JSON stringified LinkedIn person and company data, plus static company info fields.  
    - Specify expected output JSON format with "subject" and "body".  
    - Enable output parser.

17. **Add LangChain OpenRouter Chat Model Node**  
    - Name: "OpenRouter Chat Model"  
    - Connect input from "Generate Personalized Email".  
    - Configure model as `anthropic/claude-3.7-sonnet`.  
    - Provide OpenRouter API credential.

18. **Add LangChain Structured Output Parser Node**  
    - Name: "Structured Output Parser"  
    - Connect input from "OpenRouter Chat Model".  
    - Provide example schema for subject and body extraction.

19. **Connect "Structured Output Parser" output to "Generate Personalized Email" LLM node’s output parsing input**  
    - This ensures parsed output feeds back correctly.

20. **Connect "Generate Personalized Email" node output to Gmail Node**

21. **Add Gmail Node to Send Email**  
    - Name: "Gmail"  
    - Connect input from "Generate Personalized Email".  
    - Configure recipient email as `={{ $('Get Leads').item.json.Email }}`.  
    - Set subject and message body from AI output JSON.  
    - Set sender name and reply-to address from "Set Data".  
    - Use OAuth2 Gmail credentials.

22. **Connect Gmail output back to "Loop Over Items" for batch processing continuation**

23. **Add Sticky Notes**  
    - Add informative sticky notes at appropriate workflow sections explaining the logic for LinkedIn data retrieval and email sending customization.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Apify actors for Google search and LinkedIn profile scraping, which require valid API keys and may be subject to usage limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Apify platform: https://apify.com                                                                                                 |
| The AI model Claude 3.7 is accessed via OpenRouter, an API gateway for various large language models. Ensure your OpenRouter API key is valid and quota is sufficient.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | OpenRouter: https://openrouter.ai                                                                                                |
| The email generation prompt is highly customized for a specific company "AAAutomations" and can be adapted with your own company details in the "Set Data" node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                   |
| Regex filters for LinkedIn URLs assume standard URL formats and may need adjustment for future LinkedIn URL changes or edge cases.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                   |
| Gmail node requires OAuth2 credentials with sending permissions configured; ensure the Gmail account allows API access and has appropriate quotas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |                                                                                                                                   |
| The workflow assumes single-threaded batch processing for personalized emails to avoid API rate limits and ensure data integrity.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                   |
| This workflow is designed for sales outreach automation, but could be customized for other personalized email campaigns using LinkedIn data and AI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |                                                                                                                                   |

---

**Disclaimer:** The text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.