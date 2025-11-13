Convert LinkedIn Post Reactions into Qualified Leads with AI and Apify

https://n8nworkflows.xyz/workflows/convert-linkedin-post-reactions-into-qualified-leads-with-ai-and-apify-7034


# Convert LinkedIn Post Reactions into Qualified Leads with AI and Apify

---

## 1. Workflow Overview

This workflow automates the process of converting LinkedIn post reactions into qualified sales leads using AI and Apify scraping services. It is tailored for sales, marketing, and business development teams aiming to identify Ideal Customer Profiles (ICPs) from users who engage with LinkedIn posts, thereby eliminating manual prospect research.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Trigger:** Manual start for testing and initiating the data scraping.
- **1.2 Scraping Post Reactions:** Uses Apify's LinkedIn post reactions scraper to fetch users who reacted to a specific LinkedIn post.
- **1.3 Rate Limiting & Batch Processing:** Manages API rate limits and processes profiles in manageable batches with random delays to avoid detection and throttling.
- **1.4 Data Cleaning & Deduplication:** Extracts key profile fields and checks against an Airtable base to prevent processing duplicates.
- **1.5 Profile Enrichment:** Uses Apify's LinkedIn profile scraper to enrich the prospect data with detailed professional information.
- **1.6 Data Aggregation:** Gathers multiple data points per profile into structured output for further analysis.
- **1.7 AI-Powered ICP Classification:** Uses an AI model (via LangChain LLM node) to classify profiles as ICP or non-ICP, providing reasoning.
- **1.8 Data Storage & Update:** Creates new Airtable records for fresh leads and updates existing records with classification results and enriched data.
- **1.9 Workflow Safety & Documentation:** Embedded sticky notes provide guidance on usage, safety, and parameter configuration.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Trigger

**Overview:**  
Starts the workflow manually for testing or scheduled runs.

**Nodes Involved:**  
- When clicking ‘Test workflow’

**Node Details:**  
- **Type:** Manual Trigger  
- **Role:** Entry point to manually start the workflow  
- **Configuration:** No parameters; triggers workflow execution  
- **Input:** None  
- **Output:** Triggers next node "Scrape Post Reactions"  
- **Edge Cases:** None; manual initiation only  

---

### 1.2 Scraping Post Reactions

**Overview:**  
Fetches users who reacted to a specified LinkedIn post by invoking Apify's LinkedIn post reactions scraping actor.

**Nodes Involved:**  
- Scrape Post Reactions  
- Wait Rate Limit

**Node Details:**  

- **Scrape Post Reactions**  
  - **Type:** HTTP Request  
  - **Role:** Calls Apify API to scrape LinkedIn post reactions dataset  
  - **Configuration:**  
    - POST request to Apify actor endpoint  
    - Body includes `"post_url": "YOUR_POST_URN"` and `"page_number": 1`  
    - Requires Apify API token in URL query parameter  
  - **Input:** Manual trigger  
  - **Output:** JSON array of reactors to LinkedIn post  
  - **Edge Cases:**  
    - Missing or invalid token leads to auth errors  
    - API rate limits or network issues may cause failures  
    - Post URL must be valid and accessible  

- **Wait Rate Limit**  
  - **Type:** Wait node  
  - **Role:** Implements delay after scraping to respect API rate limits  
  - **Configuration:** Default wait time; no specific parameters  
  - **Input:** Output from "Scrape Post Reactions"  
  - **Output:** Passes data to next processing step  
  - **Edge Cases:** Misconfigured wait time may cause throttling or inefficiency  

---

### 1.3 Rate Limiting & Batch Processing

**Overview:**  
Processes scraped profiles in batches with random delays to avoid detection by LinkedIn and stay within API usage limits.

**Nodes Involved:**  
- Loop Over Items  
- Clean Data  
- Check Duplication  
- If  
- No Operation, do nothing  
- Create New Record  
- Random Delay  
- Random Delay Wait  
- Enrich LinkedIn Profile  
- Aggregate  
- Random Delay Generator  
- Random Delay Wait Node  
- If3  
- Edit Fields1  

**Node Details:**  

- **Loop Over Items**  
  - **Type:** SplitInBatches  
  - **Role:** Splits data into batches for sequential processing  
  - **Configuration:** Default batch size (not specified, likely 1)  
  - **Input:** Output from "Update Record" or "No Operation"  
  - **Output:** Single item batches to "Clean Data"  
  - **Edge Cases:** Large batches risk rate limits; small batches slower processing  

- **Clean Data**  
  - **Type:** Set  
  - **Role:** Extracts key profile fields from raw data for further processing  
  - **Configuration:** Sets variables: profile_urn, profile_name, profile_job_title, profile_linkedin_url from reactor object  
  - **Input:** Single profile data from batches  
  - **Output:** Cleaned JSON with profile info to "Check Duplication"  
  - **Edge Cases:** Missing fields in input JSON can cause empty outputs  

- **Check Duplication**  
  - **Type:** Airtable (search)  
  - **Role:** Checks if profile already exists in Airtable to avoid duplicates  
  - **Configuration:**  
    - Airtable base and table IDs must be configured  
    - Filter formula: matches record where Name equals profile_name  
    - Uses Airtable OAuth2 credentials  
  - **Input:** Cleaned profile data  
  - **Output:** Routes to "If" node with search results  
  - **Edge Cases:** Airtable API errors, rate limits, invalid credentials  

- **If**  
  - **Type:** If Condition  
  - **Role:** Branches workflow based on whether profile exists in Airtable  
  - **Configuration:** Checks if search result JSON is non-empty (profile exists)  
  - **Input:** From "Check Duplication"  
  - **Output:**  
    - True: Routes to "No Operation, do nothing" (skip duplicate)  
    - False: Routes to "Create New Record" (new profile)  
  - **Edge Cases:** Expression evaluation failures if input malformed  

- **No Operation, do nothing**  
  - **Type:** NoOp (No Operation)  
  - **Role:** Placeholder node to continue workflow when duplicates are found  
  - **Configuration:** None  
  - **Input:** From "If"  
  - **Output:** Passes to "Loop Over Items" to continue processing next item  

- **Create New Record**  
  - **Type:** Airtable (create)  
  - **Role:** Creates a new Airtable record for a unique profile  
  - **Configuration:**  
    - Airtable base/table IDs configured  
    - Maps profile_urn, profile_name, profile_job_title, profile_linkedin_url to respective fields  
    - Uses Airtable OAuth2 credentials  
  - **Input:** From "If" false branch  
  - **Output:** Created record details to "Random Delay"  
  - **Edge Cases:** Airtable API errors, missing required fields  

- **Random Delay** (code)  
  - **Type:** Code (JavaScript)  
  - **Role:** Generates a random delay time between 0.1 and 60 seconds (one decimal place)  
  - **Configuration:** Custom JS function returns seconds and milliseconds  
  - **Input:** From "Create New Record"  
  - **Output:** Delay parameters to "Random Delay Wait"  
  - **Edge Cases:** None significant  

- **Random Delay Wait**  
  - **Type:** Wait node  
  - **Role:** Waits for the random amount of seconds generated  
  - **Configuration:** Amount set dynamically from previous node's seconds value  
  - **Input:** From "Random Delay"  
  - **Output:** Passes to "Enrich LinkedIn Profile"  
  - **Edge Cases:** Misconfiguration could cause too short or too long delays  

- **Enrich LinkedIn Profile**  
  - **Type:** HTTP Request  
  - **Role:** Calls Apify LinkedIn Profile Scraper to enrich profile data  
  - **Configuration:**  
    - POST with JSON body containing profile_linkedin_url in array  
    - Requires Apify API token in URL query parameter  
  - **Input:** From "Random Delay Wait"  
  - **Output:** Profile enrichment JSON to "Aggregate"  
  - **Edge Cases:** API failures, invalid token, profile URL errors  

- **Aggregate**  
  - **Type:** Aggregate  
  - **Role:** Aggregates multiple fields from enriched profile data into arrays for consolidated analysis  
  - **Configuration:** Aggregates fields like fullName, jobTitle, companyName, companyIndustry, skills, about, experience titles and descriptions  
  - **Input:** From "Enrich LinkedIn Profile"  
  - **Output:** Aggregated data to "Random Delay Generator"  
  - **Edge Cases:** Missing fields cause empty arrays, inconsistent data  

- **Random Delay Generator** (code)  
  - **Type:** Code (JavaScript)  
  - **Role:** Generates random delay time as in previous random delay node  
  - **Input:** From "Aggregate"  
  - **Output:** Delay parameters to "Random Delay Wait Node"  

- **Random Delay Wait Node**  
  - **Type:** Wait  
  - **Role:** Waits for random delay time before next processing  
  - **Input:** From "Random Delay Generator"  
  - **Output:** To "If3"  

- **If3**  
  - **Type:** If Condition  
  - **Role:** Checks if aggregated data is non-empty before continuing  
  - **Configuration:** Checks if JSON object has keys and length > 0  
  - **Input:** From "Random Delay Wait Node"  
  - **Output:** Both true and false routes lead to "Edit Fields1" (possibly for error handling or downstream processing)  

- **Edit Fields1**  
  - **Type:** Set  
  - **Role:** Concatenates aggregated profile fields into a single string for AI classification input  
  - **Configuration:** Joins first elements of multiple fields into a multiline string named `linkedin_scraper_out`  
  - **Input:** From "If3"  
  - **Output:** To "AI ICP Classification"  

---

### 1.4 AI-Powered ICP Classification

**Overview:**  
Analyzes the concatenated LinkedIn profile data using an AI language model to classify whether the prospect fits the ICP with explanation.

**Nodes Involved:**  
- AI ICP Classification  
- Structured Output Parser1  
- Update Record  

**Node Details:**  

- **AI ICP Classification**  
  - **Type:** LangChain Chain LLM  
  - **Role:** Runs AI classification prompt on LinkedIn profile data  
  - **Configuration:**  
    - Custom prompt instructing AI to classify ICP with reasoning based on LinkedIn data fields provided  
    - Output parser enabled to parse structured JSON with fields `icp` (boolean) and `reasoning` (string)  
  - **Input:** From "Edit Fields1"  
  - **Output:** AI classification results to "Update Record"  
  - **Edge Cases:**  
    - AI model errors or API failures  
    - Malformed JSON output from AI may cause parsing errors  

- **Structured Output Parser1**  
  - **Type:** LangChain Structured Output Parser  
  - **Role:** Parses AI output into structured JSON format as defined by example schema  
  - **Input:** AI ICP Classification output  
  - **Output:** Parsed AI decision JSON to AI ICP Classification node (integration)  

- **Update Record**  
  - **Type:** Airtable (update)  
  - **Role:** Updates Airtable record of the prospect with AI classification results and enriched email address  
  - **Configuration:**  
    - Maps Airtable record `id` from created record  
    - Updates fields: MSP? (Boolean from AI output), Reason (AI reasoning), Email Address (from enrichment)  
  - **Input:** From "AI ICP Classification"  
  - **Output:** To "Loop Over Items" for next profile processing  
  - **Edge Cases:** Airtable API errors, mismatched record IDs, missing AI output  

---

### 1.5 Workflow Safety & Documentation

**Overview:**  
Sticky notes provide essential instructions, warnings, and contextual information about workflow usage, safety, and API configurations.

**Nodes Involved:**  
- Sticky Note1 (Overview & Purpose)  
- Sticky Note (Extract Post Reactions)  
- Sticky Note2 (Clean & Deduplicate)  
- Sticky Note3 (Profile Enrichment)  
- Sticky Note4 (AI Classification)  
- Sticky Note6 (LinkedIn Safety & Rate Limiting Warning)  

**Node Details:**  

- **Sticky Notes**  
  - **Type:** Sticky Note  
  - **Role:** Documentation and user guidance embedded within the workflow canvas  
  - **Content:** Includes setup instructions, API usage warnings, rate limiting advice, cost estimation, and pointers to official documentation links for each key node type  
  - **Position:** Strategically placed near relevant nodes for context  
  - **Edge Cases:** None; purely informational  

---

## 3. Summary Table

| Node Name               | Node Type                      | Functional Role                              | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                           |
|-------------------------|--------------------------------|----------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                | Starts the workflow manually                   |                             | Scrape Post Reactions       |                                                                                                     |
| Scrape Post Reactions    | HTTP Request                   | Scrapes LinkedIn post reactions via Apify     | When clicking ‘Test workflow’ | Wait Rate Limit             | Extract Post Reactions: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Wait Rate Limit          | Wait                          | Implements delay to respect API rate limits   | Scrape Post Reactions       | Loop Over Items             | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Loop Over Items          | SplitInBatches                | Processes profiles in batches                   | Wait Rate Limit             | Clean Data / End Loop       | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Clean Data              | Set                           | Extracts key profile fields                     | Loop Over Items             | Check Duplication           | Clean & Deduplicate: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/         |
| Check Duplication        | Airtable                      | Checks if profile already exists in Airtable   | Clean Data                  | If                         | LinkedIn Safety & Rate Limiting Warning!                                                            |
| If                      | If Condition                  | Branches workflow based on duplication check  | Check Duplication           | No Operation / Create New Record | LinkedIn Safety & Rate Limiting Warning!                                                            |
| No Operation, do nothing | NoOp                         | Skips duplicate profiles                        | If (true)                  | Loop Over Items             | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Create New Record        | Airtable                      | Adds new profile to Airtable                     | If (false)                 | Random Delay                | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Random Delay            | Code (JavaScript)             | Generates random delay time                       | Create New Record           | Random Delay Wait           | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Random Delay Wait        | Wait                         | Waits for generated random delay                  | Random Delay                | Enrich LinkedIn Profile    | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Enrich LinkedIn Profile  | HTTP Request                  | Enriches profile data via Apify LinkedIn scraper | Random Delay Wait           | Aggregate                  | Enrich LinkedIn Profiles: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Aggregate                | Aggregate                    | Aggregates multiple profile fields                | Enrich LinkedIn Profile     | Random Delay Generator      |                                                                                                     |
| Random Delay Generator   | Code (JavaScript)             | Generates random delay time                       | Aggregate                   | Random Delay Wait Node      |                                                                                                     |
| Random Delay Wait Node   | Wait                         | Waits for generated random delay                  | Random Delay Generator      | If3                        |                                                                                                     |
| If3                     | If Condition                  | Checks for presence of aggregated data            | Random Delay Wait Node      | Edit Fields1               |                                                                                                     |
| Edit Fields1            | Set                           | Creates consolidated string for AI input          | If3                        | AI ICP Classification      |                                                                                                     |
| AI ICP Classification   | LangChain Chain LLM           | Classifies profile as ICP or non-ICP with reasoning | Edit Fields1               | Update Record              | AI-Powered ICP Classification: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/ |
| Structured Output Parser1 | LangChain Structured Parser   | Parses AI output JSON                              | AI ICP Classification       | AI ICP Classification (integration) |                                                                                                     |
| Update Record           | Airtable                      | Updates Airtable record with AI results and email| AI ICP Classification       | Loop Over Items            | LinkedIn Safety & Rate Limiting Warning!                                                            |
| Sticky Note1            | Sticky Note                   | Workflow overview and purpose                      |                             |                             | 🎯 LinkedIn ICP Lead Qualification Automation                                                     |
| Sticky Note             | Sticky Note                   | Explanation of scraping LinkedIn reactions         |                             |                             | Extract Post Reactions: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Sticky Note2            | Sticky Note                   | Explanation of data cleaning and deduplication     |                             |                             | Clean & Deduplicate: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/         |
| Sticky Note3            | Sticky Note                   | Explanation of LinkedIn profile enrichment          |                             |                             | Enrich LinkedIn Profiles: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ |
| Sticky Note4            | Sticky Note                   | Explanation of AI classification                     |                             |                             | AI-Powered ICP Classification: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/ |
| Sticky Note6            | Sticky Note                   | LinkedIn safety and rate limiting warnings          |                             |                             | LinkedIn Safety & Rate Limiting Warning!                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - No parameters  
   - Connect output to next node.

2. **Add HTTP Request Node for Scraping Post Reactions:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/apimaestro~linkedin-post-reactions/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`  
   - Body (JSON):  
     ```json
     {
       "post_url": "YOUR_POST_URN",
       "page_number": 1
     }
     ```  
   - Send Body: true  
   - Connect Manual Trigger output to this node.

3. **Add Wait Node for Rate Limiting:**  
   - Type: Wait  
   - No parameters (default delay)  
   - Connect from Scrape Post Reactions.

4. **Add SplitInBatches Node:**  
   - Type: SplitInBatches  
   - Default batch size (can be adjusted for rate limits)  
   - Connect from Wait Rate Limit.

5. **Add Set Node to Clean Data:**  
   - Type: Set  
   - Add fields:  
     - `profile_urn` = `{{$json.reactor.urn}}`  
     - `profile_name` = `{{$json.reactor.name}}`  
     - `profile_job_title` = `{{$json.reactor.headline}}`  
     - `profile_linkedin_url` = `{{$json.reactor.profile_url}}`  
   - Connect from SplitInBatches.

6. **Add Airtable Search Node to Check Duplication:**  
   - Type: Airtable  
   - Operation: Search  
   - Base and Table: your Airtable base and table IDs  
   - Authentication: Airtable OAuth2  
   - Filter By Formula: `=({Name} = "{{ $json.profile_name }}")`  
   - Connect from Clean Data.

7. **Add If Node to Branch on Duplication:**  
   - Condition: Check if JSON data returned from Airtable search is non-empty (`{{$json && Object.keys($json).length > 0}}`)  
   - Connect from Check Duplication.  
   - True branch connects to No Operation node.  
   - False branch connects to Create New Record.

8. **Add No Operation Node:**  
   - Type: NoOp  
   - Connect from If (true) branch.  
   - Output connects back to SplitInBatches to continue loop.

9. **Add Airtable Create Node:**  
   - Type: Airtable  
   - Operation: Create  
   - Base and Table: your Airtable base and table IDs  
   - Map fields: URN, Name, Title, Profile URL from Set node fields  
   - Authentication: Airtable OAuth2  
   - Connect from If (false) branch.

10. **Add Code Node to Generate Random Delay:**  
    - Type: Code  
    - JavaScript code: Generate random seconds between 0.1 and 60 with one decimal place (see code in original workflow)  
    - Connect from Create New Record.

11. **Add Wait Node to Wait Random Delay:**  
    - Type: Wait  
    - Amount: `={{ $json.seconds }}` (from Code node output)  
    - Connect from Code node.

12. **Add HTTP Request Node to Enrich LinkedIn Profile:**  
    - Type: HTTP Request  
    - POST to `https://api.apify.com/v2/acts/dev_fusion~linkedin-profile-scraper/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`  
    - Body (JSON):  
      ```json
      {
        "profileUrls": ["{{ $json.profile_linkedin_url }}"]
      }
      ```  
    - Send Body: true  
    - Connect from Wait node.

13. **Add Aggregate Node:**  
    - Type: Aggregate  
    - Fields to aggregate: fullName, jobTitle, companyName, companyIndustry, topSkillsByEndorsements, about, experiences[0].title, experiences[0].subComponents[0].description, experiences[1].title, experiences[1].subComponents[0].description[0].text  
    - Connect from Enrich LinkedIn Profile.

14. **Add Second Code Node to Generate Another Random Delay:**  
    - Same as previous random delay generator code  
    - Connect from Aggregate.

15. **Add Second Wait Node for Random Delay:**  
    - Amount: `={{ $json.seconds }}`  
    - Connect from second Code node.

16. **Add If Node (If3) to Check Aggregated Data Presence:**  
    - Condition: Check if JSON has keys and length > 0 (`{{$json && Object.keys($json).length > 0}}`)  
    - Connect from second Wait node.

17. **Add Set Node (Edit Fields1) to Prepare AI Input:**  
    - Concatenate first elements of aggregated arrays into a multiline string field `linkedin_scraper_out`  
    - Connect from If3 (both true and false branches).

18. **Add LangChain Chain LLM Node for AI ICP Classification:**  
    - Type: LangChain Chain LLM  
    - Prompt: Custom prompt instructing classification of ICP based on LinkedIn data string  
    - Output Parser: Enabled with JSON schema expecting `icp` (boolean) and `reasoning` (string)  
    - Connect from Edit Fields1.

19. **Add Structured Output Parser Node:**  
    - Type: LangChain Structured Output Parser  
    - JSON Schema Example: Matches AI output format  
    - Connect to AI ICP Classification node as output parser integration.

20. **Add Airtable Update Node:**  
    - Type: Airtable  
    - Operation: Update  
    - Base and Table: your Airtable base and table IDs  
    - Map fields:  
      - `id` from created record ID  
      - `MSP?` from AI output `msp` field  
      - `Reason` from AI output `reasoning` field  
      - `Email Address` from enriched profile email (first element)  
    - Authentication: Airtable OAuth2  
    - Connect from AI ICP Classification.

21. **Connect Airtable Update Node output back to SplitInBatches node** to continue processing all profiles.

22. **Add Sticky Notes:**  
    - Add sticky notes near relevant nodes with content from the original workflow for documentation and guidance.

23. **Credentials Setup:**  
    - Set up Airtable OAuth2 credentials with access to your base and tables.  
    - Acquire Apify API token and insert into HTTP Request node URLs.  
    - Ensure AI credentials (OpenAI or compatible) configured for LangChain nodes.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| 🎯 LinkedIn ICP Lead Qualification Automation: Automatically identify and qualify ideal customer prospects from LinkedIn post reactions using AI-powered analysis. Perfect for sales and marketing teams.                                       | Sticky Note1                                                                                                                     |
| Use only cookie-free Apify actors to avoid LinkedIn detection and account suspension. Scrape a maximum of 1 page of reactions per day (50-100 profiles). Includes random delays to respect API limits and reduce risk. Budget $0.01-0.05 per profile. | Sticky Note6                                                                                                                     |
| Apify LinkedIn post reactions scraper: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/                                                                                                                        | Sticky Note                                                                                                                     |
| Clean & Deduplicate prospects to save API costs and ensure data quality using Set and Airtable nodes.                                                                                                                                           | Sticky Note2                                                                                                                     |
| Use Apify LinkedIn profile scraper to enrich prospects with detailed job titles, company data, and skills.                                                                                                                                      | Sticky Note3                                                                                                                     |
| AI classification via LangChain LLM Chain node with structured output parser for ICP determination and reasoning transparency.                                                                                                                | Sticky Note4                                                                                                                     |
| Monitor your LinkedIn and Apify usage carefully to avoid triggering anti-scraping measures. Modify AI prompt per your ICP criteria.                                                                                                           | Sticky Note6                                                                                                                     |

---

**Disclaimer:**  
The provided analysis is based exclusively on an automated n8n workflow JSON export. It adheres strictly to content policies and contains no illegal or protected content. All data processed are legal and public.

---