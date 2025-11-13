Website Intelligence & Lead Scoring with ScrapeGraphAI, HubSpot and Slack

https://n8nworkflows.xyz/workflows/website-intelligence---lead-scoring-with-scrapegraphai--hubspot-and-slack-6570


# Website Intelligence & Lead Scoring with ScrapeGraphAI, HubSpot and Slack

### 1. Workflow Overview

This workflow automates visitor tracking, company intelligence enrichment, lead scoring, CRM updating, and real-time sales alerts for website visitors. It is designed to help sales and marketing teams optimize revenue by identifying and prioritizing high-potential leads based on their behavior and company data. The workflow integrates ScrapeGraphAI for company data scraping, HubSpot CRM for contact management, and Slack for alerting sales teams.

Logical blocks:

- **1.1 Input Reception**: Captures visitor data from the website via a webhook.
- **1.2 Company Intelligence Enrichment**: Scrapes and extracts detailed company information from the visitor’s company domain using ScrapeGraphAI.
- **1.3 Visitor Data Enrichment**: Combines raw visitor data with company intelligence, analyzes behavior, and detects intent signals.
- **1.4 Lead Scoring**: Scores the enriched lead data based on multiple weighted factors and assigns qualification and priority.
- **1.5 CRM Update**: Updates or creates a contact in HubSpot CRM with enriched and scored lead data.
- **1.6 Sales Alerting**: Sends real-time notifications to the sales team via Slack for prioritized leads with actionable insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives incoming visitor tracking data from the website via an HTTP POST webhook, serving as the entry point for all subsequent processing.

- **Nodes Involved:**  
  - Webhook Trigger

- **Node Details:**  
  - **Webhook Trigger**  
    - Type: Webhook Trigger  
    - Role: Receives JSON payload with visitor tracking data  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/visitor-tracking`  
      - Expected JSON fields: visitor_id, ip_address, company_domain, page_visited, referrer, user_agent, session_duration, pages_viewed  
    - Input: External HTTP requests from website tracking scripts  
    - Output: Passes visitor data JSON to next node  
    - Edge Cases:  
      - Missing or malformed visitor data payloads  
      - Unexpected HTTP methods or paths  
      - Rate limiting or webhook security considerations  
    - Sticky Note: Describes expected data format and setup instructions  

#### 2.2 Company Intelligence Enrichment

- **Overview:**  
Uses ScrapeGraphAI to scrape the visitor’s company website domain and extract detailed company information including industry, size, services, technologies, contacts, and recent news.

- **Nodes Involved:**  
  - ScrapeGraphAI - Company Intel

- **Node Details:**  
  - **ScrapeGraphAI - Company Intel**  
    - Type: ScrapeGraphAI node (custom integration)  
    - Role: Scrapes and returns structured company data based on a schema  
    - Configuration:  
      - Dynamic URL parameter based on visitor’s company_domain or visitor_ip_domain  
      - User prompt includes JSON schema specifying fields to extract (company_name, industry, size, description, services, contact_info, social_media, key_personnel, technologies, recent_news, funding_info, competitors)  
    - Input: Visitor data JSON from webhook  
    - Output: Company intelligence JSON  
    - Credentials: Requires ScrapeGraphAI API credentials setup  
    - Edge Cases:  
      - Invalid or unavailable company domain URL  
      - API rate limits or failures  
      - Partial or missing data returned by ScrapeGraphAI  
    - Sticky Note: Explains data extracted and configuration details  

#### 2.3 Visitor Data Enrichment

- **Overview:**  
Combines raw visitor data with the scraped company intelligence into a comprehensive profile, analyzes behavioral patterns, identifies intent signals, and enriches with engagement and device type information.

- **Nodes Involved:**  
  - Visitor Enricher (Code node)

- **Node Details:**  
  - **Visitor Enricher**  
    - Type: Code node (JavaScript)  
    - Role: Merges visitor tracking and company data; calculates visitor engagement, intent signals, device type, and location  
    - Configuration:  
      - Extracts visitor and company data from inputs  
      - Functions implemented include: generateVisitorId, determineVisitType, calculateEngagement, identifyIntentSignals, getDeviceType  
      - Behavioral enrichment includes visit type (organic, referral, direct), engagement level (low/medium/high), intent signals based on visited page (pricing, demo, contact, case-study, careers)  
    - Input: Visitor data + Company data  
    - Output: Enriched visitor profile JSON with combined behavioral and company context  
    - Edge Cases:  
      - Missing or incomplete visitor or company data  
      - Unexpected user_agent formats causing device type detection failure  
    - Sticky Note: Describes enrichment steps and outputs  

#### 2.4 Lead Scoring

- **Overview:**  
Calculates a lead score based on company size, industry fit, engagement level, intent signals, and technology stack alignment. Assigns lead qualification (hot, warm, etc.) and priority, and recommends actions.

- **Nodes Involved:**  
  - Lead Scorer (Code node)

- **Node Details:**  
  - **Lead Scorer**  
    - Type: Code node (JavaScript)  
    - Role: Implements weighted scoring algorithm and lead qualification logic  
    - Configuration:  
      - Scoring factors:  
        - Company size (0-25)  
        - Industry relevance (0-20)  
        - Engagement level (0-20)  
        - Intent signals (0-25)  
        - Technology alignment (0-10)  
      - Qualification thresholds: hot (80+), warm (60-79), qualified (40-59), cold (20-39), unqualified (<20)  
      - Priority assignment based on qualification and intent signals  
      - Generates recommended sales actions and next steps  
      - Estimates deal size based on company size and industry  
    - Input: Enriched visitor profile JSON  
    - Output: Scored lead JSON including total score, qualification, priority, recommendations, and estimated deal size  
    - Edge Cases:  
      - Missing or unexpected values in enrichment data causing scoring anomalies  
      - Unusual or empty intent signals array  
    - Sticky Note: Explains scoring criteria and qualification levels  

#### 2.5 CRM Update

- **Overview:**  
Updates or creates a contact record in HubSpot CRM with the enriched lead data, including email, company info, industry, and lead scoring attributes.

- **Nodes Involved:**  
  - CRM Update (HubSpot node)

- **Node Details:**  
  - **CRM Update**  
    - Type: HubSpot CRM node  
    - Role: Synchronizes lead data into CRM contact records  
    - Configuration:  
      - Email field set dynamically from contact_info.email or fallback to visitor_id@company domain  
      - Additional fields include industry from enriched data  
      - Leverages HubSpot API credentials  
    - Input: Scored lead JSON  
    - Output: HubSpot update response data  
    - Edge Cases:  
      - Authentication or permission errors with HubSpot API  
      - Duplicate or missing email causing update failure  
      - API rate limits or timeouts  
    - Sticky Note: Details CRM integration and data sync process  

#### 2.6 Sales Alerting

- **Overview:**  
Sends real-time alert messages to a sales team Slack channel for leads classified as warm or hot, including detailed lead information and recommended sales actions.

- **Nodes Involved:**  
  - Sales Alert (Slack node)

- **Node Details:**  
  - **Sales Alert**  
    - Type: Slack node (Webhook)  
    - Role: Posts formatted alert message to a Slack channel  
    - Configuration:  
      - Message text includes visitor details, lead score, qualification, priority, intent signals, recommended actions, estimated deal size, next steps, and visit details  
      - Channel ID set via dropdown list  
      - Markdown formatting enabled  
      - Slack webhook credentials configured  
    - Input: CRM update output (scored lead data)  
    - Output: Slack post response  
    - Edge Cases:  
      - Slack API errors or invalid channel IDs  
      - Message formatting issues due to missing fields  
      - Network or webhook delivery failures  
    - Sticky Note: Explains alert triggers, content, and configuration  

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                       | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                   |
|---------------------------|--------------------------------|-------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| Webhook Trigger           | Webhook Trigger                | Receives visitor tracking data      | External HTTP request        | ScrapeGraphAI - Company Intel | Step 1: Webhook Trigger 🎣<br>Expected data format and setup instructions                     |
| ScrapeGraphAI - Company Intel | ScrapeGraphAI node            | Scrapes company website info        | Webhook Trigger             | Visitor Enricher           | Step 2: ScrapeGraphAI - Company Intel 🕵️<br>Data extracted and configuration details         |
| Visitor Enricher          | Code node                      | Enriches visitor data with company info and behavioral analysis | ScrapeGraphAI - Company Intel | Lead Scorer               | Step 3: Visitor Enricher 🔍<br>Enrichment process and output description                      |
| Lead Scorer               | Code node                      | Calculates lead score and qualification | Visitor Enricher            | CRM Update                 | Step 4: Lead Scorer 📊<br>Scoring factors and qualification levels                            |
| CRM Update                | HubSpot node                   | Updates CRM contact with lead data  | Lead Scorer                 | Sales Alert                | Step 5: CRM Update 💾<br>CRM integration and data synchronization                            |
| Sales Alert               | Slack node                    | Sends sales alert to Slack channel  | CRM Update                  |                           | Step 6: Sales Alert 🚨<br>Alert triggers and message content                                 |
| Sticky Note 1             | Sticky Note                   | Documentation                       |                             |                           | Covers Webhook Trigger (Step 1)                                                            |
| Sticky Note 2             | Sticky Note                   | Documentation                       |                             |                           | Covers ScrapeGraphAI node (Step 2)                                                          |
| Sticky Note 3             | Sticky Note                   | Documentation                       |                             |                           | Covers Visitor Enricher (Step 3)                                                             |
| Sticky Note 4             | Sticky Note                   | Documentation                       |                             |                           | Covers Lead Scorer (Step 4)                                                                  |
| Sticky Note 5             | Sticky Note                   | Documentation                       |                             |                           | Covers CRM Update (Step 5)                                                                   |
| Sticky Note 6             | Sticky Note                   | Documentation                       |                             |                           | Covers Sales Alert (Step 6)                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `visitor-tracking`  
   - Purpose: Receive visitor tracking JSON data from website scripts  
   - No credentials needed  
   - Save and activate webhook  

2. **Create ScrapeGraphAI - Company Intel Node**  
   - Type: ScrapeGraphAI node (install the ScrapeGraphAI integration in n8n)  
   - Configure API credentials for ScrapeGraphAI (API key or OAuth)  
   - Set "websiteUrl" parameter to:  
     `={{ $json.company_domain ? 'https://' + $json.company_domain : 'https://' + $json.visitor_ip_domain }}`  
   - Set "userPrompt" with JSON schema specifying required fields (company_name, industry, company_size, description, services, contact_info, social_media, key_personnel, technologies, recent_news, funding_info, competitors)  
   - Connect output of Webhook Trigger to this node  

3. **Create Visitor Enricher Node (Code)**  
   - Type: Code node (JavaScript)  
   - Configure code to:  
     - Receive visitor data and company intelligence as inputs  
     - Enrich visitor profile with combined data  
     - Implement helper functions: generateVisitorId, determineVisitType, calculateEngagement, identifyIntentSignals, getDeviceType  
     - Output enriched visitor JSON  
   - Connect output of ScrapeGraphAI node to this node  

4. **Create Lead Scorer Node (Code)**  
   - Type: Code node (JavaScript)  
   - Configure code to:  
     - Calculate lead score based on company size, industry fit, engagement, intent signals, and tech alignment  
     - Assign qualification and priority  
     - Generate recommended sales actions and next steps  
     - Estimate deal size based on company size and industry  
     - Output scored lead JSON  
   - Connect output of Visitor Enricher node to this node  

5. **Create CRM Update Node (HubSpot)**  
   - Type: HubSpot node  
   - Set up HubSpot API credentials (OAuth2 recommended)  
   - Configure fields:  
     - Email: `={{ $json.contact_info.email || $json.visitor_id + '@' + $json.company.domain }}`  
     - Industry: `={{ $json.company.industry }}`  
     - Map additional relevant lead data fields as custom properties if needed  
   - Connect output of Lead Scorer node to this node  

6. **Create Sales Alert Node (Slack)**  
   - Type: Slack node (Webhook)  
   - Configure Slack API credentials (incoming webhook or OAuth)  
   - Set target Slack channel (channelId) via dropdown or ID  
   - Use Markdown-enabled text including variables for company, lead score, qualification, intent signals, recommended actions, deal size, next steps, and visitor details  
   - Connect output of CRM Update node to this node  

7. **Connect the nodes in order:**  
   Webhook Trigger → ScrapeGraphAI - Company Intel → Visitor Enricher → Lead Scorer → CRM Update → Sales Alert  

8. **Test workflow:**  
   - Send sample visitor tracking JSON payload to webhook URL  
   - Verify company data scraping, enrichment, lead scoring, CRM update, and Slack alert delivery  
   - Handle errors or missing data by enhancing error handling or adding conditional logic if required  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow leverages ScrapeGraphAI for dynamic company data extraction based on visitor domains, providing a rich dataset for lead qualification.                                                                                                                                                                                                                                         | ScrapeGraphAI API documentation (vendor-specific)                                             |
| HubSpot integration requires correct API credentials and permission scopes to create or update contacts and set custom properties. Ensure pipeline and property mappings are aligned with your CRM setup.                                                                                                                                                                                   | HubSpot API docs: https://developers.hubspot.com/docs/api/contacts                             |
| Slack alerts are configured for high-priority leads to enable immediate sales follow-up. Customize message templates and channel IDs to fit your sales team's communication structure.                                                                                                                                                                                                    | Slack API and incoming webhook setup: https://api.slack.com/messaging/webhooks                 |
| Visitor tracking data must be collected with user consent and comply with privacy regulations (GDPR, CCPA). Ensure tracking scripts and webhook endpoints are secured.                                                                                                                                                                                                                     | Privacy compliance guidelines for web tracking (varies by jurisdiction)                       |
| This workflow can be extended with conditional branching, error handling nodes, and additional CRM or marketing automation integrations to suit specific business processes.                                                                                                                                                                                                               | n8n documentation for advanced workflow design: https://docs.n8n.io/                           |

---

**Disclaimer:**  
The text above is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.