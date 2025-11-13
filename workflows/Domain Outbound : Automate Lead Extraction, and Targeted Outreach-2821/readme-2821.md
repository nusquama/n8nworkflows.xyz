Domain Outbound : Automate Lead Extraction, and Targeted Outreach

https://n8nworkflows.xyz/workflows/domain-outbound---automate-lead-extraction--and-targeted-outreach-2821


# Domain Outbound : Automate Lead Extraction, and Targeted Outreach

### 1. Workflow Overview

**Domain Outbound Machine** is an automated n8n workflow designed to streamline the domain sales process by generating leads, extracting emails, enriching contacts, personalizing outreach, sending emails via Gmail, and tracking all interactions in an Excel file. It targets domain investors, digital marketers, and sales professionals aiming to efficiently sell domains by automating repetitive and time-consuming tasks.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts the domain name to be sold as input.
- **1.2 Query Generation:** Uses AI to generate Google Maps search queries tailored to the domain’s niche.
- **1.3 Query Formatting and Looping:** Formats and batches the queries for processing.
- **1.4 Lead Discovery via Gmail Search:** Searches Gmail for relevant URLs based on queries.
- **1.5 URL Extraction and Filtering:** Extracts URLs from search results and filters valid ones.
- **1.6 Domain and Website Data Enrichment:** Extracts domain names and enriches data using Jina.ai.
- **1.7 Email Extraction:** Scrapes emails from enriched websites.
- **1.8 Duplicate Removal:** Removes duplicate emails and leads.
- **1.9 Email Content Generation:** Uses OpenAI to create personalized email content.
- **1.10 Email Sending:** Sends personalized emails through Gmail.
- **1.11 Data Aggregation and Tracking:** Aggregates data and stores it in Google Sheets (Excel-compatible).
- **1.12 Workflow Control and Timing:** Manages batching, waiting, and looping to control flow and rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives the domain name input to start the workflow.
- **Nodes Involved:**  
  - *When clicking ‘Test workflow’* (Manual Trigger)
- **Node Details:**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing with domain input parameter (e.g., "TodayInvestment.com")  
  - Inputs: None (manual trigger)  
  - Outputs: Triggers query generation  
  - Edge Cases: Missing or invalid domain input may cause downstream failures.

#### 2.2 Query Generation

- **Overview:** Generates 100 Google Maps queries relevant to the domain’s niche using AI.
- **Nodes Involved:**  
  - *Generate queries* (OpenAI node)  
  - *fomate queries* (Code node)  
  - *Loop Over queries* (SplitInBatches node)
- **Node Details:**  
  - *Generate queries*:  
    - Type: OpenAI (LangChain)  
    - Role: Creates search queries based on domain input  
    - Config: Uses prompt to generate queries; executes once per input  
    - Outputs: Raw queries list  
    - Edge Cases: API rate limits, prompt failures  
  - *fomate queries*:  
    - Type: Code  
    - Role: Formats AI-generated queries into structured array  
    - Inputs: Raw queries  
    - Outputs: Formatted queries  
    - Edge Cases: Parsing errors if AI output format changes  
  - *Loop Over queries*:  
    - Type: SplitInBatches  
    - Role: Processes queries in batches for controlled execution  
    - Inputs: Formatted queries  
    - Outputs: Single query per batch  
    - Edge Cases: Batch size misconfiguration may cause delays or overload

#### 2.3 Lead Discovery via Gmail Search

- **Overview:** Searches Gmail for URLs related to each query to find potential leads.
- **Nodes Involved:**  
  - *Wait1* (Wait node)  
  - *Gmail search* (HTTP Request node)  
  - *Extract Urls* (Code node)  
  - *Filter urls* (Filter node)  
  - *If url is not empty* (If node)
- **Node Details:**  
  - *Wait1*:  
    - Type: Wait  
    - Role: Throttles requests to avoid Gmail API limits  
  - *Gmail search*:  
    - Type: HTTP Request  
    - Role: Queries Gmail API with search queries to find relevant emails/URLs  
    - Edge Cases: Auth errors, API limits, malformed queries  
  - *Extract Urls*:  
    - Type: Code  
    - Role: Parses Gmail search results to extract URLs  
    - Edge Cases: Parsing errors if Gmail response format changes  
  - *Filter urls*:  
    - Type: Filter  
    - Role: Removes empty or invalid URLs  
  - *If url is not empty*:  
    - Type: If  
    - Role: Branches workflow based on presence of URLs  
    - Outputs: Continues processing if URLs found; else loops or ends

#### 2.4 Domain and Website Data Enrichment

- **Overview:** Extracts domain names from URLs and enriches website data using Jina.ai.
- **Nodes Involved:**  
  - *Remove Duplicates* (RemoveDuplicates node)  
  - *Loop Over Items* (SplitInBatches node)  
  - *Extract Domain Name* (Code node)  
  - *get website with Jina.ai* (HTTP Request node)  
  - *If1* (If node)
- **Node Details:**  
  - *Remove Duplicates*:  
    - Type: RemoveDuplicates  
    - Role: Ensures unique URLs before enrichment  
  - *Loop Over Items*:  
    - Type: SplitInBatches  
    - Role: Processes URLs in batches  
  - *Extract Domain Name*:  
    - Type: Code  
    - Role: Extracts domain from URL for enrichment  
  - *get website with Jina.ai*:  
    - Type: HTTP Request  
    - Role: Calls Jina.ai API to retrieve website content and metadata for enrichment  
    - Edge Cases: API failures, timeouts  
  - *If1*:  
    - Type: If  
    - Role: Checks if enrichment data is valid before proceeding

#### 2.5 Email Extraction

- **Overview:** Extracts email addresses from enriched website data.
- **Nodes Involved:**  
  - *Loop Over Items1* (SplitInBatches node)  
  - *Extract Emails* (Code node)  
  - *Aggregate* (Aggregate node)  
  - *Split Out* (SplitOut node)  
  - *Remove Duplicates2* (RemoveDuplicates node)
- **Node Details:**  
  - *Loop Over Items1*:  
    - Type: SplitInBatches  
    - Role: Processes website data in batches for email extraction  
  - *Extract Emails*:  
    - Type: Code  
    - Role: Parses website content to find email addresses  
    - Edge Cases: False positives, missing emails  
  - *Aggregate*:  
    - Type: Aggregate  
    - Role: Combines extracted emails into a single dataset  
  - *Split Out*:  
    - Type: SplitOut  
    - Role: Splits aggregated data for further processing  
  - *Remove Duplicates2*:  
    - Type: RemoveDuplicates  
    - Role: Ensures unique email addresses before outreach

#### 2.6 Email Content Generation

- **Overview:** Generates personalized email content for each lead using AI.
- **Nodes Involved:**  
  - *Limit Markdown* (Code node)  
  - *Generate Email content* (OpenAI node)
- **Node Details:**  
  - *Limit Markdown*:  
    - Type: Code  
    - Role: Limits or formats email content markdown for compatibility  
  - *Generate Email content*:  
    - Type: OpenAI (LangChain)  
    - Role: Creates personalized email body based on enriched lead data and domain info  
    - Edge Cases: API rate limits, prompt failures

#### 2.7 Email Sending

- **Overview:** Sends personalized emails through Gmail.
- **Nodes Involved:**  
  - *Gmail1* (Gmail node)
- **Node Details:**  
  - Type: Gmail  
  - Role: Sends emails using configured Gmail OAuth2 credentials  
  - Edge Cases: Authentication errors, sending limits, invalid email addresses

#### 2.8 Data Aggregation and Tracking

- **Overview:** Stores all extracted emails and sent messages in Google Sheets for tracking.
- **Nodes Involved:**  
  - *Google Sheets* (Google Sheets node)  
  - *Code6* (Code node)
- **Node Details:**  
  - *Google Sheets*:  
    - Type: Google Sheets  
    - Role: Appends or updates rows with lead and email data  
    - Credential: Requires Google Sheets OAuth2 credentials  
  - *Code6*:  
    - Type: Code  
    - Role: Post-processing or formatting data before next steps or looping  
  - Edge Cases: API quota limits, sheet access permissions

#### 2.9 Workflow Control and Timing

- **Overview:** Controls execution flow, batching, and timing to handle API limits and ensure smooth operation.
- **Nodes Involved:**  
  - *Loop Over Items* (SplitInBatches)  
  - *Loop Over Items4* (SplitInBatches)  
  - *Wait1* (Wait)  
  - *Wait2* (Wait)
- **Node Details:**  
  - *Loop Over Items* and *Loop Over Items4*:  
    - Batch processing to control volume and pacing  
  - *Wait1* and *Wait2*:  
    - Introduce delays to avoid rate limits and manage timing between API calls

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                          | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|-------------------------|----------------------------|----------------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger             | Entry point, receives domain input     | None                         | Generate queries             |                                                                                               |
| Generate queries         | OpenAI (LangChain)          | Generates Google Maps queries           | When clicking ‘Test workflow’ | fomate queries               |                                                                                               |
| fomate queries           | Code                       | Formats AI-generated queries            | Generate queries              | Loop Over queries            |                                                                                               |
| Loop Over queries        | SplitInBatches             | Processes queries in batches             | fomate queries               | Wait1, (empty branch)        |                                                                                               |
| Wait1                   | Wait                       | Throttles Gmail search requests          | Loop Over queries             | Gmail search                |                                                                                               |
| Gmail search             | HTTP Request               | Searches Gmail for URLs based on queries | Wait1                        | Extract Urls                |                                                                                               |
| Extract Urls             | Code                       | Extracts URLs from Gmail search results  | Gmail search                 | Filter urls                 |                                                                                               |
| Filter urls              | Filter                     | Filters out empty or invalid URLs        | Extract Urls                 | If url is not empty         |                                                                                               |
| If url is not empty      | If                         | Branches based on URL presence            | Filter urls                  | Remove Duplicates, Loop Over queries |                                                                                               |
| Remove Duplicates        | RemoveDuplicates           | Removes duplicate URLs                    | If url is not empty          | Loop Over Items             |                                                                                               |
| Loop Over Items          | SplitInBatches             | Processes URLs in batches                  | Remove Duplicates            | Loop Over Items1, HTTP Request1 |                                                                                               |
| HTTP Request1            | HTTP Request               | Additional HTTP requests (unspecified)    | Loop Over Items              | Loop Over Items1            |                                                                                               |
| Loop Over Items1         | SplitInBatches             | Processes website data batches             | Loop Over Items, HTTP Request1 | Aggregate, Extract Emails   |                                                                                               |
| Aggregate               | Aggregate                  | Aggregates extracted emails                | Loop Over Items1             | Split Out                   |                                                                                               |
| Split Out               | SplitOut                   | Splits aggregated data                      | Aggregate                   | Remove Duplicates2           |                                                                                               |
| Remove Duplicates2       | RemoveDuplicates           | Removes duplicate emails                    | Split Out                   | Loop Over Items4             |                                                                                               |
| Loop Over Items4         | SplitInBatches             | Processes leads in batches                   | Remove Duplicates2           | Loop Over queries, Extract Domain Name |                                                                                               |
| Extract Domain Name      | Code                       | Extracts domain from URL                      | Loop Over Items4             | get website with Jina.ai     |                                                                                               |
| get website with Jina.ai | HTTP Request               | Enriches website data via Jina.ai API       | Extract Domain Name          | If1                         |                                                                                               |
| If1                     | If                         | Checks validity of enrichment data          | get website with Jina.ai     | Loop Over Items4, Limit Markdown |                                                                                               |
| Limit Markdown          | Code                       | Formats email content markdown               | If1                         | Generate Email content       |                                                                                               |
| Generate Email content  | OpenAI (LangChain)          | Creates personalized email content          | Limit Markdown               | Gmail1                      |                                                                                               |
| Gmail1                  | Gmail                      | Sends personalized emails                    | Generate Email content       | Google Sheets               |                                                                                               |
| Google Sheets           | Google Sheets              | Stores leads and sent emails                  | Gmail1                      | Code6                       |                                                                                               |
| Code6                   | Code                       | Post-processing and looping control           | Google Sheets               | Wait2                       |                                                                                               |
| Wait2                   | Wait                       | Controls pacing between batches               | Code6                       | Loop Over Items4             |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Accept domain input (e.g., "TodayInvestment.com") to start workflow.

2. **Add OpenAI Node to Generate Queries**  
   - Type: OpenAI (LangChain)  
   - Configure prompt to generate 100 Google Maps queries based on domain niche.  
   - Connect Manual Trigger output to this node.

3. **Add Code Node to Format Queries**  
   - Type: Code  
   - Write JavaScript to parse and format AI-generated queries into an array.  
   - Connect OpenAI output to this node.

4. **Add SplitInBatches Node to Loop Over Queries**  
   - Type: SplitInBatches  
   - Configure batch size (e.g., 1) to process queries one at a time.  
   - Connect Code node output to this node.

5. **Add Wait Node (Wait1)**  
   - Type: Wait  
   - Purpose: Throttle requests to avoid Gmail API limits.  
   - Connect SplitInBatches output to Wait node.

6. **Add HTTP Request Node for Gmail Search**  
   - Type: HTTP Request  
   - Configure to call Gmail API with current query to search for relevant emails/URLs.  
   - Use OAuth2 credentials for Gmail.  
   - Connect Wait node output to this node.

7. **Add Code Node to Extract URLs**  
   - Type: Code  
   - Parse Gmail search response to extract URLs.  
   - Connect HTTP Request output to this node.

8. **Add Filter Node to Remove Empty URLs**  
   - Type: Filter  
   - Condition: URL field is not empty.  
   - Connect Code node output to this node.

9. **Add If Node to Branch on URL Presence**  
   - Type: If  
   - Condition: URL exists.  
   - Connect Filter node output to this node.

10. **Add RemoveDuplicates Node to Remove Duplicate URLs**  
    - Type: RemoveDuplicates  
    - Connect If node’s true branch to this node.

11. **Add SplitInBatches Node to Loop Over URLs**  
    - Type: SplitInBatches  
    - Connect RemoveDuplicates output to this node.

12. **Add HTTP Request Node for Additional Calls (Optional)**  
    - Type: HTTP Request  
    - Connect SplitInBatches output to this node if needed for further data fetching.

13. **Add SplitInBatches Node to Process Website Data**  
    - Type: SplitInBatches  
    - Connect HTTP Request and/or previous SplitInBatches output to this node.

14. **Add Code Node to Extract Emails from Website Data**  
    - Type: Code  
    - Parse website content to extract emails.  
    - Connect SplitInBatches output to this node.

15. **Add Aggregate Node to Combine Emails**  
    - Type: Aggregate  
    - Connect Code node output to this node.

16. **Add SplitOut Node to Split Aggregated Data**  
    - Type: SplitOut  
    - Connect Aggregate output to this node.

17. **Add RemoveDuplicates Node to Remove Duplicate Emails**  
    - Type: RemoveDuplicates  
    - Connect SplitOut output to this node.

18. **Add SplitInBatches Node to Loop Over Leads**  
    - Type: SplitInBatches  
    - Connect RemoveDuplicates output to this node.

19. **Add Code Node to Extract Domain Name from URL**  
    - Type: Code  
    - Extract domain from each lead’s URL.  
    - Connect SplitInBatches output to this node.

20. **Add HTTP Request Node to Enrich Website Data via Jina.ai**  
    - Type: HTTP Request  
    - Configure to call Jina.ai API with domain info.  
    - Connect Code node output to this node.

21. **Add If Node to Check Enrichment Validity**  
    - Type: If  
    - Condition: Enrichment data is valid.  
    - Connect HTTP Request output to this node.

22. **Add Code Node to Limit/Format Email Markdown**  
    - Type: Code  
    - Format AI-generated email content for compatibility.  
    - Connect If node’s false branch to this node.

23. **Add OpenAI Node to Generate Personalized Email Content**  
    - Type: OpenAI (LangChain)  
    - Use enriched data to generate personalized email body.  
    - Connect Code node output to this node.

24. **Add Gmail Node to Send Emails**  
    - Type: Gmail  
    - Configure with Gmail OAuth2 credentials.  
    - Connect OpenAI node output to this node.

25. **Add Google Sheets Node to Store Leads and Emails**  
    - Type: Google Sheets  
    - Configure with Google Sheets OAuth2 credentials and target spreadsheet.  
    - Connect Gmail node output to this node.

26. **Add Code Node for Post-Processing**  
    - Type: Code  
    - Optional: Format or prepare data for next loop or logging.  
    - Connect Google Sheets output to this node.

27. **Add Wait Node (Wait2) for Pacing**  
    - Type: Wait  
    - Controls pacing between batches to avoid rate limits.  
    - Connect Code node output to this node.

28. **Connect Wait2 output back to Loop Over Leads**  
    - To continue processing next batch of leads.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automates domain sales lead generation, email extraction, personalized outreach, and tracking.       | Workflow description and use case summary.                                                      |
| Uses OpenAI (LangChain) nodes for query and email content generation.                                          | Requires OpenAI API credentials.                                                                |
| Integrates Gmail API for searching and sending emails; requires OAuth2 credentials with appropriate scopes.   | Gmail OAuth2 setup needed.                                                                       |
| Uses Google Sheets for tracking leads and sent emails; requires Google Sheets OAuth2 credentials.             | Google Sheets API setup needed.                                                                  |
| Jina.ai API used for website content enrichment; ensure API access and credentials are configured.             | External enrichment service integration.                                                        |
| Includes wait nodes to handle API rate limits and pacing.                                                     | Important for avoiding throttling and errors.                                                   |
| Sticky notes in the workflow provide additional context and instructions (not included here).                 | Refer to workflow UI for sticky note details.                                                   |
| Workflow designed for scalability: batch sizes and wait times can be adjusted based on API limits and volume. |                                                                                                 |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and maintaining the **Domain Outbound Machine** workflow in n8n. It covers all nodes, their roles, configurations, and integration points to facilitate advanced usage and troubleshooting.