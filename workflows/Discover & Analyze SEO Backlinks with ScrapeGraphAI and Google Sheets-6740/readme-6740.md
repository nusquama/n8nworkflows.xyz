Discover & Analyze SEO Backlinks with ScrapeGraphAI and Google Sheets

https://n8nworkflows.xyz/workflows/discover---analyze-seo-backlinks-with-scrapegraphai-and-google-sheets-6740


# Discover & Analyze SEO Backlinks with ScrapeGraphAI and Google Sheets

### 1. Workflow Overview

This workflow automates the discovery, analysis, and outreach process for SEO backlink opportunities, targeting digital marketing and SEO professionals aiming to improve their website authority and content strategy. It integrates AI-powered scraping, data analysis, and contact discovery with Google Sheets tracking and automated email outreach.

The workflow is logically divided into the following blocks:

- **1.1 Weekly Trigger**: Initiates the workflow on a scheduled weekly basis.
- **1.2 Competitor Backlink Scraping**: Uses AI to scrape backlink data from competitor websites.
- **1.3 Backlink Opportunity Analysis**: Processes scraped data to score and prioritize backlink opportunities.
- **1.4 High-Priority Filtering**: Filters opportunities to focus only on high-priority backlinks.
- **1.5 Contact Information Discovery**: Scrapes websites for relevant contact info related to the backlink opportunities.
- **1.6 Data Merging and Outreach Preparation**: Combines backlink and contact data, prepares personalized outreach content.
- **1.7 Google Sheets Tracking**: Appends the compiled data into a Google Sheets document for monitoring and team collaboration.
- **1.8 Automated Outreach Email Campaign**: Sends personalized emails to contacts with valid email addresses to initiate partnership discussions.

---

### 2. Block-by-Block Analysis

#### 2.1 Weekly Trigger

- **Overview:**  
  Initiates the entire workflow automatically on a weekly schedule to ensure fresh backlink opportunities are discovered regularly.

- **Nodes Involved:**  
  - Weekly Schedule Trigger

- **Node Details:**  
  - **Weekly Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Executes once every week (default weekly interval; no specific day/time shown but can be configured)  
    - Inputs: None (trigger node)  
    - Outputs: Triggers the "AI-Powered Competitor Backlink Scraper" node  
    - Edge Cases: Misconfiguration of schedule (e.g., timezone errors) or disabled workflow would prevent execution.  
    - Notes: Best practice is to schedule during business hours for timely follow-up.

---

#### 2.2 Competitor Backlink Scraping

- **Overview:**  
  Scrapes competitor backlink profiles using AI to extract detailed backlink data including URLs, anchor texts, domain authority scores, and contextual information.

- **Nodes Involved:**  
  - AI-Powered Competitor Backlink Scraper

- **Node Details:**  
  - **AI-Powered Competitor Backlink Scraper**  
    - Type: ScrapeGraphAI node  
    - Configuration:  
      - User prompt instructs the AI to find backlinks to competitor websites, extracting structured data fields such as source_url, target_url, anchor_text, domain_authority, page_title, and context.  
      - Website URL input is set statically to "https://ahrefs.com/backlink-checker/", a known backlink analysis tool.  
    - Inputs: Triggered by Weekly Schedule Trigger  
    - Outputs: Provides JSON array of backlinks for the next stage  
    - Version Dependency: Requires ScrapeGraphAI community node installed and configured with API credentials  
    - Edge Cases: API limits, scrape failures, invalid competitor URLs, or no backlinks found  
    - Notes: Uses AI to interpret page context, improving data quality.

---

#### 2.3 Backlink Opportunity Analysis

- **Overview:**  
  Applies a scoring algorithm to each scraped backlink opportunity to prioritize based on domain authority, anchor text relevance, context length, and a randomized relevance factor.

- **Nodes Involved:**  
  - Backlink Opportunity Analyzer

- **Node Details:**  
  - **Backlink Opportunity Analyzer**  
    - Type: Code Node (JavaScript)  
    - Configuration:  
      - Parses backlink data from previous node  
      - Scores each backlink with weights: 40% domain authority, 20 points anchor text, 15 points context, 25 points random factor  
      - Assigns priority labels: High (>70), Medium (40-70), Low (<40)  
      - Outputs enriched backlink objects with scoring and priority  
    - Inputs: Output of AI-Powered Competitor Backlink Scraper  
    - Outputs: Enriched backlink data to both High Priority Opportunity Filter and Google Sheets Tracker  
    - Edge Cases: Missing backlink fields, parsing errors, empty backlink lists  
    - Notes: Includes timestamp for analysis date.

---

#### 2.4 High-Priority Filtering

- **Overview:**  
  Filters the analyzed backlink opportunities to pass only those with a "High" priority for further contact information discovery and outreach.

- **Nodes Involved:**  
  - High Priority Opportunity Filter

- **Node Details:**  
  - **High Priority Opportunity Filter**  
    - Type: Filter Node  
    - Configuration: Condition checks if `priority` field equals "High"  
    - Inputs: Backlink Opportunity Analyzer output  
    - Outputs: Passes high priority backlinks to Contact Information Finder  
    - Edge Cases: No high priority backlinks found results in no downstream processing  
    - Notes: Thresholds are adjustable in the scoring code node.

---

#### 2.5 Contact Information Discovery

- **Overview:**  
  Uses AI to discover contact information such as emails, contact forms, social media, and team member details for the source URLs of high priority backlinks.

- **Nodes Involved:**  
  - AI-Powered Contact Information Finder

- **Node Details:**  
  - **AI-Powered Contact Information Finder**  
    - Type: ScrapeGraphAI node  
    - Configuration:  
      - User prompt instructs the AI to find detailed contact info in a structured JSON format  
      - Website URL dynamically set from the source_url of the backlink opportunity  
    - Inputs: High Priority Opportunity Filter output  
    - Outputs: Provides structured contact info JSON for merging  
    - Version Dependency: Requires ScrapeGraphAI node and API credentials  
    - Edge Cases: Websites without contact info, scraping failures, incomplete data  
    - Notes: AI recognizes multiple contact formats and social media handles.

---

#### 2.6 Data Merging and Outreach Preparation

- **Overview:**  
  Merges backlink opportunity data with contact info, adds outreach metadata, and generates personalized subject lines and messages for the email campaign.

- **Nodes Involved:**  
  - Data Merger and Outreach Preparation

- **Node Details:**  
  - **Data Merger and Outreach Preparation**  
    - Type: Code Node (JavaScript)  
    - Configuration:  
      - Receives backlink data and contact data as inputs  
      - Combines them into a single JSON object per opportunity  
      - Adds outreach tracking fields: status, attempts, last contact date, notes  
      - Generates personalized email subject and message based on page title and anchor text  
    - Inputs: Two inputs - from Contact Information Finder and High Priority Opportunity Filter (via Filter and ScrapeGraphAI nodes)  
    - Outputs: Merged data to Email Available Filter node  
    - Edge Cases: Missing contact info fields, data mismatch, null values  
    - Notes: Prepares data for both storage and outreach.

---

#### 2.7 Google Sheets Tracking

- **Overview:**  
  Appends all backlink opportunity data, including contact and outreach fields, into a Google Sheets document for collaborative tracking and management.

- **Nodes Involved:**  
  - Google Sheets Opportunity Tracker

- **Node Details:**  
  - **Google Sheets Opportunity Tracker**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append rows  
      - Sheet Name: "Backlink_Opportunities"  
      - Document ID: Configured via credentials or URL (not specified in JSON)  
      - Columns mapped automatically from input JSON fields including source_url, target_url, anchor_text, domain_authority, opportunity_score, priority, emails, outreach_status, suggested_subject  
    - Inputs: Backlink Opportunity Analyzer output (all priorities are stored)  
    - Outputs: None (terminal node)  
    - Edge Cases: Authentication errors, quota limits, invalid document ID, schema mismatch  
    - Notes: Enables campaign management, progress tracking, and reporting.

---

#### 2.8 Automated Outreach Email Campaign

- **Overview:**  
  Sends personalized outreach emails to prospects with valid email addresses, aiming to initiate partnership discussions based on the backlink opportunity.

- **Nodes Involved:**  
  - Email Available Filter  
  - Automated Outreach Email Campaign

- **Node Details:**  
  - **Email Available Filter**  
    - Type: Filter Node  
    - Configuration: Checks if `emails` array length is greater than zero  
    - Inputs: Data Merger and Outreach Preparation output  
    - Outputs: Passes records with valid emails to Email Send node  
    - Edge Cases: No emails found leads to no outreach  
  - **Automated Outreach Email Campaign**  
    - Type: Email Send node  
    - Configuration:  
      - Subject set dynamically from `suggested_subject`  
      - Message body taken from `outreach_message` in previous node (implied, but not explicitly shown)  
      - SMTP credentials required (not specified in JSON)  
      - Additional options: replyTo blank, allows attachments, forbids unauthorized certs  
    - Inputs: Email Available Filter output  
    - Outputs: None (terminal node)  
    - Edge Cases: SMTP authentication failures, invalid email addresses, rate limits, bounces  
    - Notes: Requires preconfigured SMTP credentials and compliance with email regulations.

---

### 3. Summary Table

| Node Name                         | Node Type                        | Functional Role                       | Input Node(s)                           | Output Node(s)                           | Sticky Note                                                                                             |
|----------------------------------|---------------------------------|-------------------------------------|---------------------------------------|-----------------------------------------|-------------------------------------------------------------------------------------------------------|
| Weekly Schedule Trigger           | Schedule Trigger                 | Initiates weekly workflow run       | None                                  | AI-Powered Competitor Backlink Scraper  | Step 1: Weekly Schedule Trigger ⏰ - Explains schedule config, benefits, and best practices             |
| AI-Powered Competitor Backlink Scraper | ScrapeGraphAI Node              | Scrapes competitor backlinks        | Weekly Schedule Trigger                | Backlink Opportunity Analyzer            | Step 2: AI-Powered Competitor Backlink Scraper 🕷️ - Details scraping logic and AI usage                |
| Backlink Opportunity Analyzer    | Code Node                       | Scores backlink opportunities       | AI-Powered Competitor Backlink Scraper | High Priority Opportunity Filter, Google Sheets Opportunity Tracker | Step 3: Backlink Opportunity Analyzer 📊 - Describes scoring factors and priority classification        |
| High Priority Opportunity Filter | Filter Node                    | Filters high priority backlinks     | Backlink Opportunity Analyzer         | AI-Powered Contact Information Finder    | Step 4: High Priority Opportunity Filter 🎯 - Focuses outreach on best prospects                        |
| AI-Powered Contact Information Finder | ScrapeGraphAI Node              | Finds contact info on backlink sites | High Priority Opportunity Filter      | Data Merger and Outreach Preparation     | Step 5: AI-Powered Contact Information Finder 👥 - Extracts emails, social profiles, team members       |
| Data Merger and Outreach Preparation | Code Node                       | Merges data and prepares outreach   | AI-Powered Contact Information Finder | Email Available Filter                    | Step 6: Data Merger and Outreach Preparation 🔗 - Combines data, generates outreach messages            |
| Email Available Filter            | Filter Node                    | Filters records with emails         | Data Merger and Outreach Preparation  | Automated Outreach Email Campaign         |                                                                                                       |
| Google Sheets Opportunity Tracker | Google Sheets Node              | Stores all backlink opportunities   | Backlink Opportunity Analyzer         | None                                    | Step 7: Google Sheets Opportunity Tracker 📈 - Stores data for campaign management                      |
| Automated Outreach Email Campaign | Email Send Node                | Sends personalized outreach emails | Email Available Filter                 | None                                    | Step 8: Automated Outreach Email Campaign 📧 - Sends emails, requires SMTP credentials                  |
| Step 1 - Schedule Trigger Info   | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 1 sticky note content                                                                         |
| Step 2 - Competitor Scraper Info | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 2 sticky note content                                                                         |
| Step 3 - Opportunity Analyzer Info | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 3 sticky note content                                                                         |
| Step 4 - Priority Filter Info    | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 4 sticky note content                                                                         |
| Step 5 - Contact Finder Info     | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 5 sticky note content                                                                         |
| Step 6 - Data Merger Info        | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 6 sticky note content                                                                         |
| Step 7 - Sheets Tracker Info     | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 7 sticky note content                                                                         |
| Step 8 - Email Campaign Info     | Sticky Note                    | Documentation                       | None                                  | None                                    | See Step 8 sticky note content                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Weekly Schedule Trigger**  
   - Node Type: Schedule Trigger  
   - Set interval to execute once every week (adjust day/time/timezone as needed).

2. **Add AI-Powered Competitor Backlink Scraper node (ScrapeGraphAI)**  
   - Node Type: ScrapeGraphAI  
   - Set `websiteUrl` to "https://ahrefs.com/backlink-checker/"  
   - User Prompt:  
     ```
     Find all the backlinks pointing to competitor websites in our niche. Extract the following information for each backlink: { "source_url": "https://example.com/page", "target_url": "https://competitor.com", "anchor_text": "relevant keyword", "domain_authority": "65", "page_title": "Page Title", "context": "surrounding text context" }
     ```  
   - Connect output of Weekly Schedule Trigger to this node.  
   - Configure ScrapeGraphAI credentials with your API key.

3. **Add Backlink Opportunity Analyzer (Code Node)**  
   - Node Type: Code  
   - Paste the provided JavaScript code that:  
     - Parses backlinks array from input  
     - Scores each backlink with weighted factors  
     - Assigns priority labels (High/Medium/Low)  
     - Adds analyzed date in `YYYY-MM-DD` format  
   - Connect output of ScrapeGraphAI node to this node.

4. **Add High Priority Opportunity Filter**  
   - Node Type: Filter  
   - Set condition: `$json.priority` equals "High" (case sensitive)  
   - Connect output of Backlink Opportunity Analyzer to this filter node.

5. **Add AI-Powered Contact Information Finder (ScrapeGraphAI)**  
   - Node Type: ScrapeGraphAI  
   - Set `websiteUrl` to expression: `{{$json.source_url}}` (dynamic per opportunity)  
   - User Prompt:  
     ```
     Find contact information on this website including email addresses, contact forms, social media profiles, and any team member information. Return data in this format: { "emails": ["contact@example.com", "info@example.com"], "contact_forms": ["https://example.com/contact"], "social_media": { "twitter": "@handle", "linkedin": "company-page" }, "team_members": [{ "name": "John Doe", "role": "Content Manager", "email": "john@example.com" }] }
     ```  
   - Connect output of High Priority Opportunity Filter to this node.  
   - Configure ScrapeGraphAI credentials with the API key.

6. **Add Data Merger and Outreach Preparation (Code Node)**  
   - Node Type: Code  
   - Configure to take two inputs:  
     - Input 1: Backlink opportunity data (from AI-Powered Contact Information Finder input 0)  
     - Input 2: Contact info data (from AI-Powered Contact Information Finder input 1)  
   - Use provided JavaScript code to merge backlink and contact data, add outreach metadata, and generate personalized email subject and message.  
   - Connect output of AI-Powered Contact Information Finder to this node.

7. **Add Email Available Filter**  
   - Node Type: Filter  
   - Condition: `$json.emails.length` > 0 (ensuring there is at least one email address)  
   - Connect output of Data Merger and Outreach Preparation node to this filter.

8. **Add Automated Outreach Email Campaign (Email Send Node)**  
   - Node Type: Email Send  
   - Set Subject to expression: `{{$json.suggested_subject}}`  
   - Configure message body to use `outreach_message` from input JSON (either via parameters or dynamic content injection)  
   - Configure SMTP credentials for your email server.  
   - Connect output of Email Available Filter to this node.

9. **Add Google Sheets Opportunity Tracker**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Sheet Name: "Backlink_Opportunities"  
   - Map input fields automatically (including source_url, target_url, anchor_text, domain_authority, opportunity_score, priority, emails, outreach_status, suggested_subject)  
   - Connect output of Backlink Opportunity Analyzer node to this node (to store all opportunities regardless of priority)  
   - Configure Google Sheets credentials and specify the document ID or URL.

10. **Ensure all nodes are connected as per the flow:**  
    - Weekly Schedule Trigger → AI-Powered Competitor Backlink Scraper  
    - AI-Powered Competitor Backlink Scraper → Backlink Opportunity Analyzer  
    - Backlink Opportunity Analyzer → High Priority Opportunity Filter AND Google Sheets Opportunity Tracker  
    - High Priority Opportunity Filter → AI-Powered Contact Information Finder  
    - AI-Powered Contact Information Finder → Data Merger and Outreach Preparation  
    - Data Merger and Outreach Preparation → Email Available Filter  
    - Email Available Filter → Automated Outreach Email Campaign  

11. **Verify credentials for ScrapeGraphAI, Google Sheets, and SMTP Email nodes are properly set up.**

12. **Test the workflow with sample data to validate scraping, scoring, filtering, merging, storage, and email sending.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| ScrapeGraphAI nodes require API keys and community node installation in n8n.                                                                                        | ScrapeGraphAI official documentation or community resources                                    |
| Google Sheets node requires OAuth2 credentials with access to the target spreadsheet.                                                                                | Google Cloud Console and n8n credentials setup                                                 |
| Email Send node requires SMTP credentials; ensure compliance with email sending regulations (CAN-SPAM, GDPR).                                                      | SMTP provider documentation and legal guidelines                                              |
| Weekly scheduling recommended during business hours to enable timely manual follow-up if needed.                                                                    | Sticky note Step 1 content                                                                     |
| The scoring algorithm balances domain authority, anchor text relevance, content context, and randomness to identify promising backlink opportunities dynamically.    | Sticky note Step 3 content                                                                     |
| Filtering focuses outreach on high-priority backlinks to maximize resource efficiency and ROI.                                                                       | Sticky note Step 4 content                                                                     |
| Contact discovery uses AI to extract diverse contact formats, increasing outreach success chances.                                                                  | Sticky note Step 5 content                                                                     |
| Data merging enhances outreach personalization, improving engagement rates.                                                                                          | Sticky note Step 6 content                                                                     |
| Google Sheets tracking facilitates team collaboration and campaign analytics.                                                                                       | Sticky note Step 7 content                                                                     |
| Automated emails use dynamic subjects and messages personalized per prospect, increasing response likelihood.                                                      | Sticky note Step 8 content                                                                     |

---

This completes a detailed, structured reference for the "Discover & Analyze SEO Backlinks with ScrapeGraphAI and Google Sheets" workflow, enabling advanced users and automation agents to understand, reproduce, and maintain the process effectively.