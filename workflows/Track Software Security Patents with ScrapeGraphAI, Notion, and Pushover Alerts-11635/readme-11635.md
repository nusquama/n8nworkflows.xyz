Track Software Security Patents with ScrapeGraphAI, Notion, and Pushover Alerts

https://n8nworkflows.xyz/workflows/track-software-security-patents-with-scrapegraphai--notion--and-pushover-alerts-11635


# Track Software Security Patents with ScrapeGraphAI, Notion, and Pushover Alerts

### 1. Workflow Overview

This workflow automates the weekly tracking of new software security-related patents by combining web scraping, data enrichment, storage, and mobile alerts. It targets R&D teams and patent analysts who need timely insight into recent patent filings involving vulnerabilities or cybersecurity. The workflow is structured into six logical blocks:

- **1.1 Schedule & Query URL Generation**: Triggers the workflow weekly and builds search URLs based on predefined keywords.
- **1.2 AI-Powered Scraping**: Uses ScrapeGraphAI to extract recent patent metadata in a structured JSON format from the search URLs.
- **1.3 Data Flattening & Batching**: Converts array results into individual patent items for granular processing and throttles requests to avoid rate limits.
- **1.4 Enrichment & Filtering**: Enriches patent data via the PatentView API, normalizes fields, detects security relevance, and deduplicates.
- **1.5 Storage in Notion**: Saves clean and enriched patent records into a collaborative Notion database.
- **1.6 Mobile Notifications**: Sends Pushover alerts for patents flagged as important (security related).

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Query URL Generation

**Overview:**  
This block starts the workflow on a weekly schedule and builds a list of patent search URLs using technology-related keywords.

**Nodes Involved:**  
- Weekly Trigger  
- Build Query URLs  
- Split URLs  

**Node Details:**

- **Weekly Trigger**  
  - Type: Schedule Trigger  
  - Role: Fires the workflow every 168 hours (weekly), at a fixed time (default 08:00 Monday)  
  - Configuration: Interval set to 168 hours  
  - Inputs: None (trigger node)  
  - Outputs: List of items to downstream nodes  
  - Failures: None typical unless internal scheduler issues occur  

- **Build Query URLs**  
  - Type: Code  
  - Role: Constructs Google Patents search URLs for each keyword  
  - Configuration: Hardcoded keywords array with values: ‘software vulnerability’, ‘cybersecurity’, ‘memory safety’  
  - Key expression: Maps keywords to objects containing a URL and its keyword label  
  - Input: Triggered by Weekly Trigger  
  - Output: Array of `{ url: <search-url>, keyword: <keyword> }` objects  
  - Failures: Code exceptions if malformed keywords; no dynamic input handling  

- **Split URLs**  
  - Type: Split In Batches  
  - Role: Processes one URL at a time to throttle requests and avoid HTTP 429 errors  
  - Configuration: Default batch size = 1  
  - Input: List of URLs from Build Query URLs  
  - Output: Single URL objects forwarded sequentially  
  - Failures: Batch processing errors if input is empty or malformed  

---

#### 2.2 AI-Powered Scraping

**Overview:**  
This block uses ScrapeGraphAI’s LLM-driven scraping to extract up to 20 recent patents from each search URL, producing clean patent metadata in JSON.

**Nodes Involved:**  
- Scrape Patent Listings  
- Flatten Patent List  

**Node Details:**

- **Scrape Patent Listings**  
  - Type: ScrapeGraphAI  
  - Role: Loads the patent search page URL and extracts patent data using a natural language prompt  
  - Configuration:  
    - User prompt requests extraction of 20 most recent patents with fields: title, applicationNumber, publicationDate, abstract, url  
    - Website URL dynamically set from the incoming JSON field `$json.url`  
  - Inputs: Single URL object from Split URLs  
  - Outputs: JSON with a `patents` array containing patent records  
  - Failures:  
    - API key or network errors  
    - Changes in webpage structure that complicate extraction despite LLM resilience  
    - Rate limits by ScrapeGraphAI service  

- **Flatten Patent List**  
  - Type: Code  
  - Role: Converts the patents array into individual items, one per patent, for downstream processing  
  - Configuration: JavaScript code iterates over all input items, flattening the `patents` array into single patent JSON objects  
  - Inputs: Output array from Scrape Patent Listings  
  - Outputs: Single patent objects per item  
  - Failures: If input lacks a patents array or is malformed, code will skip such entries  

---

#### 2.3 Data Flattening & Batching

**Overview:**  
This mini-block batches individual patent items for sequential enrichment API calls to avoid overloading external services.

**Nodes Involved:**  
- Split Patents  

**Node Details:**

- **Split Patents**  
  - Type: Split In Batches  
  - Role: Processes one patent record at a time to throttle PatentView API calls  
  - Configuration: Default batch size = 1  
  - Inputs: Individual patent records from Flatten Patent List  
  - Outputs: Single patent record forwarded sequentially  
  - Failures: Empty or invalid inputs can halt or skip batches  

---

#### 2.4 Enrichment & Filtering

**Overview:**  
This block enriches each patent with additional metadata from the PatentView API, normalizes the data fields, and flags security-relevant patents.

**Nodes Involved:**  
- Patent Details API  
- Transform & Deduplicate  

**Node Details:**

- **Patent Details API**  
  - Type: HTTP Request  
  - Role: Queries PatentView API to retrieve patent classifications, inventors, and assignee details using the application number  
  - Configuration:  
    - URL dynamically built using `applicationNumber` from each patent  
    - Returns fields: patent_number, inventors, cpcs, assignee_organization  
  - Inputs: Single patent record from Split Patents  
  - Outputs: JSON with enriched patent data  
  - Failures:  
    - Network errors, API downtime, or rate limiting  
    - Missing or invalid application number causing empty responses  

- **Transform & Deduplicate**  
  - Type: Code  
  - Role: Merges raw scraped data and enrichment data, normalizes fields (e.g., ISO-8601 dates), strips line breaks, trims strings, and flags patents as `important` if keywords like “security” or CPC subclass `G06F` appear  
  - Configuration:  
    - Uses regex to detect vulnerability-related terms in abstract  
    - Creates an `important` boolean flag accordingly  
    - Extracts inventor names concatenated from first and last names  
  - Inputs: Enriched patent data from Patent Details API  
  - Outputs: Cleaned, deduplicated patent records with enriched metadata and importance flag  
  - Failures: If enrichment data is missing or malformed, might produce incomplete output; regex evaluation errors possible but unlikely  

---

#### 2.5 Storage in Notion

**Overview:**  
Stores each processed patent record into a Notion database configured for collaborative review and filtering.

**Nodes Involved:**  
- Store in Notion  

**Node Details:**

- **Store in Notion**  
  - Type: Notion  
  - Role: Inserts or updates patent records in a specified Notion database with mapped properties  
  - Configuration:  
    - Database ID must be configured by user  
    - Maps patent title to “Name” field, application number, publication date, URL, and CPC codes to corresponding Notion properties  
  - Inputs: Transformed patent records from Transform & Deduplicate  
  - Outputs: Confirmation of database entry creation/update  
  - Failures:  
    - Authorization errors if credentials or database ID are incorrect  
    - API limits or network errors  
    - Incorrect property mapping causing data loss  

---

#### 2.6 Mobile Notifications

**Overview:**  
Sends push notifications for patents flagged as important to the user’s mobile device via Pushover.

**Nodes Involved:**  
- Important Patent?  
- Pushover Alert  

**Node Details:**

- **Important Patent?**  
  - Type: IF  
  - Role: Filters patents, passing only those where `important` flag is true  
  - Configuration: Checks boolean condition `$json.important === true`  
  - Inputs: Patent records from Store in Notion  
  - Outputs: Only important patents forwarded  
  - Failures: Misconfiguration of condition could block alerts  

- **Pushover Alert**  
  - Type: Pushover  
  - Role: Sends a mobile push notification containing the patent title and publication date  
  - Configuration:  
    - Message template combines title and publication date  
    - Priority set to 0 (normal) by default  
    - Requires Pushover credentials configured in n8n  
  - Inputs: Important patent records from Important Patent? node  
  - Outputs: Confirmation of alert sent  
  - Failures:  
    - Credential errors  
    - Network or Pushover service downtime  

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                 |
|-----------------------|--------------------------|----------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow Overview     | Sticky Note              | Describes overall workflow purpose     | None                   | None                    | Explains weekly schedule, scraping via ScrapeGraphAI, enrichment, Notion storage, and Pushover alerts      |
| Section – Data Collection | Sticky Note            | Describes data collection block        | None                   | None                    | Notes on weekly trigger, URL building, and batching to avoid rate limits                                  |
| Section – Scraping    | Sticky Note              | Describes AI-powered scraping           | None                   | None                    | Details ScrapeGraphAI prompt and flattening logic                                                          |
| Section – Enrichment  | Sticky Note              | Describes API enrichment and filtering | None                   | None                    | Explains enrichment, normalization, and importance flagging                                               |
| Section – Storage     | Sticky Note              | Describes Notion storage                | None                   | None                    | Explains storage in Notion database and collaboration benefits                                            |
| Section – Notification | Sticky Note             | Describes mobile alerts                 | None                   | None                    | Describes IF filter and Pushover alert with throttling for manageable notifications                        |
| Weekly Trigger         | Schedule Trigger         | Triggers workflow weekly                | None                   | Build Query URLs        |                                                                                                            |
| Build Query URLs       | Code                     | Builds patent search URLs               | Weekly Trigger          | Split URLs              |                                                                                                            |
| Split URLs             | Split In Batches         | Batches URLs to avoid rate limits       | Build Query URLs        | Scrape Patent Listings  |                                                                                                            |
| Scrape Patent Listings | ScrapeGraphAI            | Extracts patent metadata                 | Split URLs              | Flatten Patent List     |                                                                                                            |
| Flatten Patent List    | Code                     | Flattens patents array into individual records | Scrape Patent Listings | Split Patents           |                                                                                                            |
| Split Patents          | Split In Batches         | Batches patents one by one               | Flatten Patent List     | Patent Details API      |                                                                                                            |
| Patent Details API     | HTTP Request             | Enriches patent data via PatentView API| Split Patents           | Transform & Deduplicate |                                                                                                            |
| Transform & Deduplicate| Code                     | Normalizes, deduplicates, flags important | Patent Details API    | Store in Notion         |                                                                                                            |
| Store in Notion        | Notion                   | Stores patent records in Notion database| Transform & Deduplicate | Important Patent?       |                                                                                                            |
| Important Patent?      | IF                       | Filters important patents for alerts   | Store in Notion         | Pushover Alert          |                                                                                                            |
| Pushover Alert         | Pushover                 | Sends mobile push notification          | Important Patent?       | None                    |                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set interval to 168 hours (1 week) to run every Monday at 08:00 by default.

2. **Add a Code node “Build Query URLs”:**  
   - Define an array of keywords: `['software vulnerability', 'cybersecurity', 'memory safety']`.  
   - Map each keyword to a Google Patents search URL with URL-encoded keyword parameters.  
   - Output: array of objects with `{ url: <search-url>, keyword: <keyword> }`.

3. **Add a Split In Batches node “Split URLs”:**  
   - Connect from “Build Query URLs”.  
   - Configure to batch size 1 for throttling.

4. **Add a ScrapeGraphAI node “Scrape Patent Listings”:**  
   - Connect from “Split URLs”.  
   - Configure with the user prompt:  
     _“Extract the 20 most recent patents from this search result. Return an array called patents where each element contains: title, applicationNumber, publicationDate, abstract, url.”_  
   - Set website URL to `={{ $json.url }}`.  
   - Set credentials with your ScrapeGraphAI API key.

5. **Add a Code node “Flatten Patent List”:**  
   - Connect from “Scrape Patent Listings”.  
   - Code to flatten the `patents` array into individual patent objects.

6. **Add a Split In Batches node “Split Patents”:**  
   - Connect from “Flatten Patent List”.  
   - Batch size 1 to throttle API calls.

7. **Add an HTTP Request node “Patent Details API”:**  
   - Connect from “Split Patents”.  
   - Configure URL to query PatentView API:  
     ```
     https://www.patentsview.org/api/patents/query?q={"patent_number":"{{$json.applicationNumber}}"}&f=["patent_number","inventors","cpcs","assignee_organization"]
     ```  
   - Use GET method, no authentication needed unless API key required.  
   - Handle JSON response.

8. **Add a Code node “Transform & Deduplicate”:**  
   - Connect from “Patent Details API”.  
   - Code merges scraped and enriched data, normalizes dates to ISO-8601, trims strings, and sets `important` flag if abstract or CPC codes indicate security relevance.  
   - Extract inventor names as full names.

9. **Add a Notion node “Store in Notion”:**  
   - Connect from “Transform & Deduplicate”.  
   - Configure with your Notion credentials and target database ID.  
   - Map properties:  
     - Name ← title  
     - Application Number ← applicationNumber  
     - Publication Date ← publicationDate (date type)  
     - URL ← url  
     - CPC ← cpc (multi-select or text)  

10. **Add an IF node “Important Patent?”:**  
    - Connect from “Store in Notion”.  
    - Condition: `{{$json.important}}` equals true.

11. **Add a Pushover node “Pushover Alert”:**  
    - Connect from “Important Patent?” (true output).  
    - Configure message: `{{$json.title}} — published {{$json.publicationDate}}`.  
    - Set priority to 0 (normal).  
    - Configure Pushover credentials (application token, user key).

12. **Activate the workflow:**  
    - Test by running manually once to verify mappings and outputs.  
    - Enable to run weekly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow runs weekly to avoid overloading patent office servers and APIs, respecting rate limits and politeness.                     | Workflow Overview sticky note                                                                                   |
| ScrapeGraphAI uses natural language prompts instead of brittle selectors, providing resilience to UI changes on patent sites.       | Section – Scraping sticky note                                                                                   |
| PatentView API enriches scraped data with inventor and classification info; no authentication required for public endpoints.         | Section – Enrichment sticky note                                                                                 |
| Notion database enables collaborative filtering, comments, and audit trails for patent records.                                      | Section – Storage sticky note                                                                                    |
| Pushover alerts ensure researchers only get notified of critical security-related patents, reducing noise.                           | Section – Notification sticky note                                                                               |
| Setup requires API keys/credentials for ScrapeGraphAI, Notion, and Pushover; database ID must be set in the Notion node.             | Workflow Overview sticky note                                                                                     |
| Keywords can be customized in the “Build Query URLs” code node to tailor patent searches to specific technologies or domains.       | Workflow Overview sticky note                                                                                     |
| Example prompt for ScrapeGraphAI extraction: “Extract the 20 most recent patents... with title, applicationNumber, publicationDate...” | Scrape Patent Listings node configuration                                                                        |

---

**Disclaimer:**  
This document is a professional analysis derived exclusively from an n8n automated workflow. It complies with all content policies and does not contain illegal, offensive, or protected material. All data handled is legal and publicly available.