Generate, score, and send B2B leads with Google Places, Airtable, and email

https://n8nworkflows.xyz/workflows/generate--score--and-send-b2b-leads-with-google-places--airtable--and-email-18801


# Generate, score, and send B2B leads with Google Places, Airtable, and email

### 1. Workflow Overview

This workflow is an automated end-to-end B2B lead generation, enrichment, scoring, and multi-channel outreach system. Its primary purpose is to discover local business leads via Google Places, normalize and store them in an Airtable CRM, scrape their websites for contact details and metadata, perform optional performance audits using the Google PageSpeed Insights API, calculate a rule-based opportunity score to prioritize leads (Hot, Warm, Low), draft customized outreach communications, and execute human-approved multi-channel messaging via SMTP (email), WhatsApp, and Instagram.

The workflow logic is categorized into the following functional blocks:
- **1.1 Lead Discovery & Ingestion:** Initializes search parameters, queries the Google Places API in batches, de-duplicates results, and upserts them into Airtable.
- **1.2 Website Scraping & Validation:** Verifies domain existence, screens out social-media-only domains, scrapes valid websites for metadata, and flags unreachable or missing websites in the CRM.
- **1.3 PageSpeed Performance Auditing:** Performs mobile and desktop performance/SEO audits via the Google PageSpeed API and writes metrics or failure logs back to Airtable.
- **1.4 Lead Scoring & Routing:** Retrieves unscored leads from Airtable, computes opportunity scores and priorities, and routes them to generate tailored outreach messaging drafts.
- **1.5 Approval Queue Initialization:** Identifies new leads awaiting review and initializes an approval queue in the CRM.
- **1.6 Multi-Channel Outreach Dispatch:** Processes approved leads, checks channel eligibility, crafts professional messages, and dispatches communications via SMTP email, WhatsApp, and Instagram worker endpoints with error handling and status tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Discovery & Ingestion
- **Overview:** Initializes discovery configurations, constructs a search matrix of target locations and search terms, queries the Google Places API in batches, de-duplicates records, and synchronizes the leads with the Airtable CRM using the unique `place_id`.
- **Nodes Involved:** `Manual Lead Discovery Trigger`, `Scheduled Lead Discovery`, `Set Lead Discovery Config`, `Check Lead Discovery Config`, `Build Search Matrix`, `Batch Process Searches`, `Post Places Search`, `Handle Places API Error`, `Set Search Context`, `Wait 2 Seconds`, `Split Places`, `Deduplicate by ID`, `Set Lead Details`, `Sync Lead with CRM by ID`.
- **Node Details:**
  - **Manual Lead Discovery Trigger / Scheduled Lead Discovery**: Triggers the initial discovery phase manually or on a recurring schedule.
  - **Set Lead Discovery Config**: Sets environment variables and configurations for target locations, search terms, and max results.
  - **Check Lead Discovery Config**: Validates that required discovery parameters are present.
  - **Build Search Matrix**: Generates combinations of locations and search keywords using a Code node.
  - **Batch Process Searches**: Splits the search matrix into controlled batches using a Split In Batches node.
  - **Post Places Search**: HTTP Request node querying the Google Places API v1. Handles pagination and error output. Potential failure: API rate limits or invalid keys.
  - **Handle Places API Error**: Stops execution and logs an error if the Places API fails.
  - **Set Search Context / Wait 2 Seconds**: Adds throttling delays between requests to respect API rate limits.
  - **Split Places**: Split Out node separating individual place items from the API response payload.
  - **Deduplicate by ID**: Removes duplicate leads based on the unique Google `place_id`.
  - **Set Lead Details**: Formats and maps place attributes to standard CRM fields.
  - **Sync Lead with CRM by ID**: Airtable node upserting the lead record using `place_id` as the matching key.

#### 2.2 Website Scraping & Validation
- **Overview:** Evaluates whether each lead possesses a valid, non-social-media website, fetches the site HTML, extracts contact metadata (emails, phones, page title, description), and records results or failure states in the CRM.
- **Nodes Involved:** `Verify Website Exists`, `Check Valid Website`, `Flag No Website in CRM`, `Flag Social Media Only`, `GET Lead Website`, `Check Website Response`, `Flag Website Unreachable`, `Extract from Website`, `Set Research Fields`, `Update CRM with Research`.
- **Node Details:**
  - **Verify Website Exists**: IF node checking if a website URL is present on the lead record.
  - **Check Valid Website**: IF node filtering out generic social media profile URLs (e.g., Facebook, Instagram links used as websites).
  - **Flag No Website in CRM / Flag Social Media Only / Flag Website Unreachable**: Airtable nodes updating the CRM status when a website is missing, restricted to social media, or returns HTTP errors.
  - **GET Lead Website**: HTTP Request node fetching the target website's HTML with retry logic and error output mapping.
  - **Check Website Response**: IF node checking if the HTTP status code indicates a successful response.
  - **Extract from Website**: HTML node extracting emails, phone numbers, titles, and descriptions from raw HTML.
  - **Set Research Fields**: Set node organizing extracted metadata into structured attributes.
  - **Update CRM with Research**: Airtable node updating the CRM record with extracted contact and page details.

#### 2.3 PageSpeed Performance Auditing
- **Overview:** Checks if PageSpeed auditing is enabled, requests mobile and desktop performance metrics from the Google PageSpeed Insights API, and records performance scores or failure logs in the CRM.
- **Nodes Involved:** `Check PageSpeed Enabled`, `GET PageSpeed Mobile`, `Check NO_FCP`, `Set Mobile PageSpeed Data`, `Record Mobile PageSpeed in CRM`, `GET PageSpeed Desktop`, `Set Desktop PageSpeed Data`, `Record Desktop PageSpeed in CRM`, `Set Failed PageSpeed Data`, `Log PageSpeed Fail in CRM`, `Set Failed Desktop PageSpeed`, `Log Desktop PageSpeed Fail`.
- **Node Details:**
  - **Check PageSpeed Enabled**: IF node verifying feature flag settings.
  - **GET PageSpeed Mobile / GET PageSpeed Desktop**: HTTP Request nodes querying the Google PageSpeed Insights API with retry handling and wait intervals.
  - **Check NO_FCP**: IF node checking for missing First Contentful Paint metrics in mobile results.
  - **Set Mobile PageSpeed Data / Set Desktop PageSpeed Data**: Set nodes formatting performance scores and core web vitals.
  - **Record Mobile PageSpeed in CRM / Record Desktop PageSpeed in CRM**: Airtable nodes saving performance metrics to the CRM.
  - **Set Failed PageSpeed Data / Set Failed Desktop PageSpeed / Log PageSpeed Fail in CRM / Log Desktop PageSpeed Fail**: Set and Airtable nodes capturing and logging audit API failures or missing metrics.

#### 2.4 Lead Scoring & Routing
- **Overview:** Pulls unscored leads from Airtable on a schedule, calculates rule-based opportunity scores and priority levels (Hot, Warm, Low), determines recommended services, and saves the evaluation results back to the CRM before routing them for outreach drafting.
- **Nodes Involved:** `Scheduled Lead Scoring`, `Set Scoring Config`, `Verify Scoring Config`, `Fetch Scoring Leads`, `Compute Opportunity Score`, `Create Lead Recommendation`, `Log Scoring in CRM`, `Route Leads by Priority`, `Prepare High Priority Outreach`, `Save High Priority Outreach`, `Prepare Moderate Priority Outreach`, `Save Moderate Priority Outreach`, `Set Low Priority Outreach`, `Save Low Priority to CRM`.
- **Node Details:**
  - **Scheduled Lead Scoring**: Schedule Trigger node initiating the scoring batch.
  - **Set Scoring Config / Verify Scoring Config**: Configuration and validation nodes for scoring weights and rules.
  - **Fetch Scoring Leads**: Airtable node retrieving records marked as unscored.
  - **Compute Opportunity Score / Create Lead Recommendation**: Code nodes executing scoring algorithms and generating tailored service pitches.
  - **Log Scoring in CRM**: Airtable node saving computed scores, priorities, and recommendations.
  - **Route Leads by Priority**: Switch node splitting the flow into Hot, Warm, and Low priority branches.
  - **Prepare High Priority Outreach / Prepare Moderate Priority Outreach / Set Low Priority Outreach**: Custom preparation and set nodes building tailored outreach drafts and nurturing states.
  - **Save High Priority Outreach / Save Moderate Priority Outreach / Save Low Priority to CRM**: Airtable nodes updating the CRM with generated drafts and priorities.

#### 2.5 Approval Queue Initialization
- **Overview:** Fetches leads awaiting approval from Airtable, formats the approval queue data, and stores the queue state in the CRM.
- **Nodes Involved:** `Fetch Awaiting Approval Leads`, `Set Approval Queue Data`, `Store Approval Queue in CRM`.
- **Node Details:**
  - **Fetch Awaiting Approval Leads**: Airtable node querying records awaiting review.
  - **Set Approval Queue Data**: Set node formatting queue attributes.
  - **Store Approval Queue in CRM**: Airtable node updating queue records.

#### 2.6 Multi-Channel Outreach Dispatch
- **Overview:** Triggered by schedule, fetches approved leads from Airtable, crafts professional multi-channel messages, verifies channel eligibility, and dispatches approved communications via SMTP email, WhatsApp, and Instagram worker endpoints with real-time status updates (`Sending`, `Sent`, `Failed`).
- **Nodes Involved:** `Scheduled Outreach Processing`, `Set Outreach Config`, `Check Outreach Config`, `Fetch Approved Send Leads`, `Set Approved Send Details`, `Log Approval Timestamp`, `Craft Professional Message`, `Confirm Email Eligibility`, `Flag Email as Sending`, `Dispatch Approved Email`, `Flag Email as Sent`, `Flag Email as Failed`, `Confirm WhatsApp Eligibility`, `Organize WhatsApp Send Details`, `Verify WhatsApp Eligibility`, `Flag WhatsApp as Sending`, `Dispatch WhatsApp Message`, `Flag WhatsApp as Sent`, `Flag WhatsApp as Failed`, `Arrange WhatsApp Manual Review`, `Log WhatsApp Manual Review`, `Confirm Instagram Eligibility`, `Create Instagram Send Details`, `Verify Instagram Eligibility`, `Flag Instagram as Sending`, `Dispatch Instagram DM`, `Flag Instagram as Sent`, `Flag Instagram as Failed`, `Organize Instagram Manual Review`, `Log Instagram Manual Review`.
- **Node Details:**
  - **Scheduled Outreach Processing**: Schedule Trigger node initiating outreach execution.
  - **Set Outreach Config / Check Outreach Config**: Configuration and validation nodes for sender identities and worker URLs.
  - **Fetch Approved Send Leads / Set Approved Send Details / Log Approval Timestamp**: Airtable nodes fetching approved records, setting dispatch variables, and logging timestamps.
  - **Craft Professional Message**: Code node generating customized outreach copy.
  - **Confirm Email / WhatsApp / Instagram Eligibility**: IF nodes verifying whether each channel is enabled and has valid recipient contact data.
  - **Flag ... as Sending / Sent / Failed**: HTTP Request and Airtable nodes updating per-channel delivery statuses in real time.
  - **Dispatch Approved Email**: Send Email node (SMTP) delivering outbound emails.
  - **Dispatch WhatsApp Message / Dispatch Instagram DM**: HTTP Request nodes communicating with external worker URLs for messaging APIs.
  - **Arrange / Log Manual Review**: Fallback nodes handling leads lacking direct messaging channels by logging review tasks.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | stickyNote | Placeholder note | — | — | |
| Split Places | splitOut | Splits Google Places array | Split Places | Deduplicate by ID | |
| Set Lead Details | set | Formats place attributes | Deduplicate by ID | Sync Lead with CRM by ID | |
| Sync Lead with CRM by ID | airtable | Upserts lead in CRM | Set Lead Details | Verify Website Exists | |
| Verify Website Exists | if | Checks if website exists | Sync Lead with CRM by ID | Check Valid Website, Flag No Website in CRM | |
| GET Lead Website | httpRequest | Fetches website HTML | Check Valid Website | Check Website Response, Flag Website Unreachable | |
| Check Website Response | if | Validates HTTP response | GET Lead Website | Extract from Website, Flag Website Unreachable | |
| Extract from Website | html | Extracts emails & metadata | Check Website Response | Set Research Fields | |
| Set Research Fields | set | Structures research data | Extract from Website | Update CRM with Research | |
| Update CRM with Research | airtable | Updates CRM with research | Set Research Fields | Check PageSpeed Enabled | |
| Flag No Website in CRM | airtable | Flags missing website in CRM | Verify Website Exists | — | |
| Flag Website Unreachable | airtable | Flags unreachable website | GET Lead Website, Check Website Response | — | |
| Check Valid Website | if | Filters out social media links | Verify Website Exists | GET Lead Website, Flag Social Media Only | |
| Flag Social Media Only | airtable | Flags social-only domains | Check Valid Website | — | |
| GET PageSpeed Mobile | httpRequest | Requests mobile PageSpeed audit | Check PageSpeed Enabled | Set Mobile PageSpeed Data, Check NO_FCP | |
| Check NO_FCP | if | Checks for missing mobile FCP | GET PageSpeed Mobile | GET PageSpeed Desktop, Set Failed PageSpeed Data | |
| GET PageSpeed Desktop | httpRequest | Requests desktop PageSpeed audit | Check NO_FCP | Set Desktop PageSpeed Data, Set Failed Desktop PageSpeed | |
| Set Desktop PageSpeed Data | set | Formats desktop metrics | GET PageSpeed Desktop | Record Desktop PageSpeed in CRM | |
| Set Mobile PageSpeed Data | set | Formats mobile metrics | GET PageSpeed Mobile | Record Mobile PageSpeed in CRM | |
| Record Mobile PageSpeed in CRM | airtable | Saves mobile metrics to CRM | Set Mobile PageSpeed Data | — | |
| Record Desktop PageSpeed in CRM | airtable | Saves desktop metrics to CRM | Set Desktop PageSpeed Data | — | |
| Set Failed PageSpeed Data | set | Formats mobile failure data | Check NO_FCP | Log PageSpeed Fail in CRM | |
| Log PageSpeed Fail in CRM | airtable | Logs mobile audit failure | Set Failed PageSpeed Data | — | |
| Set Failed Desktop PageSpeed | set | Formats desktop failure data | GET PageSpeed Desktop | Log Desktop PageSpeed Fail | |
| Log Desktop PageSpeed Fail | airtable | Logs desktop audit failure | Set Failed Desktop PageSpeed | — | |
| Deduplicate by ID | removeDuplicates | Removes duplicate leads | Split Places | Set Lead Details | |
| Set Lead Discovery Config | set | Sets discovery configuration | Scheduled Lead Discovery, Manual Lead Discovery Trigger | Check Lead Discovery Config | |
| Build Search Matrix | code | Generates search location matrix | Check Lead Discovery Config | Batch Process Searches | |
| Batch Process Searches | splitInBatches | Batches search execution | Build Search Matrix, Wait 2 Seconds | Split Places, Post Places Search | |
| Wait 2 Seconds | wait | Throttles API requests | Set Search Context | Batch Process Searches | |
| Post Places Search | httpRequest | Queries Google Places API | Batch Process Searches | Set Search Context, Handle Places API Error | |
| Handle Places API Error | stopAndError | Stops execution on API failure | Post Places Search | — | |
| Check Lead Discovery Config | code | Validates discovery config | Set Lead Discovery Config | Build Search Matrix | |
| Manual Lead Discovery Trigger | manualTrigger | Manual discovery trigger | — | Set Lead Discovery Config | |
| Scheduled Lead Discovery | scheduleTrigger | Scheduled discovery trigger | — | Set Lead Discovery Config | |
| Set Search Context | set | Sets search context variables | Post Places Search | Wait 2 Seconds | |
| Check PageSpeed Enabled | if | Checks if PageSpeed is enabled | Update CRM with Research | GET PageSpeed Mobile | |
| Fetch Scoring Leads | airtable | Retrieves unscored leads | Verify Scoring Config | Compute Opportunity Score | |
| Compute Opportunity Score | code | Computes lead opportunity score | Fetch Scoring Leads | Create Lead Recommendation | |
| Create Lead Recommendation | code | Generates messaging angle | Compute Opportunity Score | Log Scoring in CRM | |
| Log Scoring in CRM | airtable | Saves scoring data to CRM | Create Lead Recommendation | Route Leads by Priority | |
| Route Leads by Priority | switch | Routes leads by priority level | Log Scoring in CRM | Prepare High Priority Outreach, Prepare Moderate Priority Outreach, Set Low Priority Outreach | |
| Prepare High Priority Outreach | code | Drafts high-priority outreach | Route Leads by Priority | Save High Priority Outreach | |
| Save High Priority Outreach | airtable | Saves high-priority draft | Prepare High Priority Outreach | — | |
| Prepare Moderate Priority Outreach | code | Drafts moderate-priority outreach | Route Leads by Priority | Save Moderate Priority Outreach | |
| Save Moderate Priority Outreach | airtable | Saves moderate-priority draft | Prepare Moderate Priority Outreach | — | |
| Set Low Priority Outreach | set | Sets low-priority nurture state | Route Leads by Priority | Save Low Priority to CRM | |
| Save Low Priority to CRM | airtable | Saves low-priority state | Set Low Priority Outreach | — | |
| Set Scoring Config | set | Sets scoring configuration | Scheduled Lead Scoring | Verify Scoring Config | |
| Verify Scoring Config | code | Validates scoring configuration | Set Scoring Config | Fetch Scoring Leads | |
| Scheduled Lead Scoring | scheduleTrigger | Scheduled scoring trigger | — | Set Scoring Config | |
| Fetch Awaiting Approval Leads | airtable | Fetches review queue leads | Check Outreach Config | Set Approval Queue Data | |
| Set Approval Queue Data | set | Formats approval queue data | Fetch Awaiting Approval Leads | Store Approval Queue in CRM | |
| Store Approval Queue in CRM | airtable | Stores approval queue in CRM | Set Approval Queue Data | — | |
| Fetch Approved Send Leads | airtable | Fetches approved send leads | Check Outreach Config | Set Approved Send Details | |
| Set Approved Send Details | set | Formats approved send details | Fetch Approved Send Leads | Log Approval Timestamp, Craft Professional Message | |
| Log Approval Timestamp | airtable | Logs approval timestamp | Set Approved Send Details | — | |
| Dispatch Approved Email | emailSend | Sends outbound email | Flag Email as Sending | Flag Email as Sent, Flag Email as Failed | |
| Craft Professional Message | code | Crafts outreach message | Set Approved Send Details | Confirm Email Eligibility, Confirm WhatsApp Eligibility, Confirm Instagram Eligibility | |
| Flag Email as Sent | airtable | Updates CRM email status | Dispatch Approved Email | — | |
| Create Instagram Send Details | set | Formats Instagram send data | Confirm Instagram Eligibility | Verify Instagram Eligibility | |
| Organize Instagram Manual Review | set | Organizes Instagram review | Verify Instagram Eligibility | Log Instagram Manual Review | |
| Log Instagram Manual Review | httpRequest | Logs Instagram manual review | Organize Instagram Manual Review | — | |
| Dispatch Instagram DM | httpRequest | Sends Instagram direct message | Flag Instagram as Sending | Flag Instagram as Sent, Flag Instagram as Failed | |
| Flag Instagram as Sent | httpRequest | Updates Instagram sent status | Dispatch Instagram DM | — | |
| Organize WhatsApp Send Details | set | Formats WhatsApp send data | Confirm WhatsApp Eligibility | Verify WhatsApp Eligibility | |
| Verify WhatsApp Eligibility | if | Checks WhatsApp number validity | Organize WhatsApp Send Details, Verify WhatsApp Eligibility | Flag WhatsApp as Sending, Arrange WhatsApp Manual Review | |
| Dispatch WhatsApp Message | httpRequest | Sends WhatsApp message | Flag WhatsApp as Sending | Flag WhatsApp as Sent, Flag WhatsApp as Failed | |
| Flag WhatsApp as Sent | httpRequest | Updates WhatsApp sent status | Dispatch WhatsApp Message | — | |
| Arrange WhatsApp Manual Review | set | Organizes WhatsApp review | Verify WhatsApp Eligibility | Log WhatsApp Manual Review | |
| Log WhatsApp Manual Review | httpRequest | Logs WhatsApp manual review | Arrange WhatsApp Manual Review | — | |
| Flag Email as Sending | httpRequest | Updates email sending status | Confirm Email Eligibility | Dispatch Approved Email | |
| Flag Email as Failed | airtable | Updates email failure status | Dispatch Approved Email | — | |
| Flag WhatsApp as Failed | httpRequest | Updates WhatsApp failure status | Dispatch WhatsApp Message | — | |
| Flag Instagram as Failed | httpRequest | Updates Instagram failure status | Dispatch Instagram DM | — | |
| Scheduled Outreach Processing | scheduleTrigger | Scheduled outreach trigger | — | Set Outreach Config | |
| Confirm Email Eligibility | if | Checks email eligibility | Craft Professional Message | Flag Email as Sending | |
| Confirm WhatsApp Eligibility | if | Checks WhatsApp eligibility | Craft Professional Message | Organize WhatsApp Send Details | |
| Confirm Instagram Eligibility | if | Checks Instagram eligibility | Craft Professional Message | Create Instagram Send Details | |
| Flag WhatsApp as Sending | httpRequest | Updates WhatsApp sending status | Verify WhatsApp Eligibility | Dispatch WhatsApp Message | |
| Flag Instagram as Sending | httpRequest | Updates Instagram sending status | Verify Instagram Eligibility | Dispatch Instagram DM | |
| Set Outreach Config | set | Sets outreach configuration | Scheduled Outreach Processing | Check Outreach Config | |
| Check Outreach Config | code | Validates outreach configuration | Set Outreach Config | Fetch Awaiting Approval Leads, Fetch Approved Send Leads | |
| Verify Instagram Eligibility | if | Checks Instagram handle validity | Create Instagram Send Details, Verify Instagram Eligibility | Flag Instagram as Sending, Organize Instagram Manual Review | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these step-by-step instructions:

#### Phase 1: Discovery Setup
1. Create a **Manual Trigger** node and a **Schedule Trigger** node (disabled by default) as alternative entry points.
2. Add a **Set** node (`Set Lead Discovery Config`) to define base parameters (target locations, search keywords, max results).
3. Add a **Code** node (`Check Lead Discovery Config`) to validate the configuration parameters.
4. Add a **Code** node (`Build Search Matrix`) to create search combinations.
5. Add a **Split In Batches** node (`Batch Process Searches`) to paginate through the search queries.
6. Add an **HTTP Request** node (`Post Places Search`) targeting the Google Places API v1. Configure error handling to continue on error.
7. Add a **Set** node (`Set Search Context`) and a **Wait** node (`Wait 2 Seconds`) to throttle requests. Connect back to the batch node.
8. Add a **Split Out** node (`Split Places`) to separate individual place items.
9. Add a **Remove Duplicates** node (`Deduplicate by ID`) keyed on `place_id`.
10. Add a **Set** node (`Set Lead Details`) to format lead data and an **Airtable** node (`Sync Lead with CRM by ID`) to upsert records using `place_id`.

#### Phase 2: Website Scraping & Validation
1. Add an **IF** node (`Verify Website Exists`) to check if a website URL is populated.
2. Add an **IF** node (`Check Valid Website`) to filter out social-media-only URLs.
3. Add **Airtable** nodes (`Flag No Website in CRM`, `Flag Social Media Only`, `Flag Website Unreachable`) for respective failure branches.
4. Add an **HTTP Request** node (`GET Lead Website`) with retry logic enabled (3 tries, 2000ms wait) and error output enabled.
5. Add an **IF** node (`Check Website Response`) to verify successful HTTP status codes.
6. Add an **HTML** node (`Extract from Website`) to parse emails, phone numbers, and page metadata.
7. Add a **Set** node (`Set Research Fields`) and an **Airtable** node (`Update CRM with Research`) to save extracted data.

#### Phase 3: PageSpeed Auditing
1. Add an **IF** node (`Check PageSpeed Enabled`) to check feature flags.
2. Add an **HTTP Request** node (`GET PageSpeed Mobile`) targeting the Google PageSpeed Insights API.
3. Add an **IF** node (`Check NO_FCP`) to verify mobile metrics.
4. Add **Set** and **Airtable** nodes (`Set Mobile PageSpeed Data`, `Record Mobile PageSpeed in CRM`, `Set Failed PageSpeed Data`, `Log PageSpeed Fail in CRM`) to handle success and failure states.
5. Repeat a similar pattern for desktop audits using `GET PageSpeed Desktop`, `Set Desktop PageSpeed Data`, `Record Desktop PageSpeed in CRM`, `Set Failed Desktop PageSpeed`, and `Log Desktop PageSpeed Fail`.

#### Phase 4: Scoring & Routing
1. Add a **Schedule Trigger** node (`Scheduled Lead Scoring`, disabled by default).
2. Add **Set** and **Code** nodes (`Set Scoring Config`, `Verify Scoring Config`) to load scoring rules.
3. Add an **Airtable** node (`Fetch Scoring Leads`) to pull unscored leads.
4. Add **Code** nodes (`Compute Opportunity Score`, `Create Lead Recommendation`) to calculate scores and generate tailored recommendations.
5. Add an **Airtable** node (`Log Scoring in CRM`) to save scoring results.
6. Add a **Switch** node (`Route Leads by Priority`) to route leads into Hot, Warm, and Low priority branches.
7. For each priority branch, add a **Code** or **Set** node (`Prepare High Priority Outreach`, `Prepare Moderate Priority Outreach`, `Set Low Priority Outreach`) followed by corresponding **Airtable** nodes to save outreach drafts.

#### Phase 5: Approval Queue & Outreach Dispatch
1. Add **Airtable** nodes (`Fetch Awaiting Approval Leads`, `Store Approval Queue in CRM`) via a **Set** node to manage the approval queue.
2. Add a **Schedule Trigger** node (`Scheduled Outreach Processing`, disabled by default).
3. Add **Set** and **Code** nodes (`Set Outreach Config`, `Check Outreach Config`) to initialize outreach settings.
4. Add an **Airtable** node (`Fetch Approved Send Leads`) to pull approved records.
5. Add a **Set** node (`Set Approved Send Details`) and an **Airtable** node (`Log Approval Timestamp`).
6. Add a **Code** node (`Craft Professional Message`) to generate outbound text.
7. Add **IF** nodes (`Confirm Email Eligibility`, `Confirm WhatsApp Eligibility`, `Confirm Instagram Eligibility`) to verify channel criteria.
8. For Email: Add HTTP request/Airtable status flags and an **Email Send (SMTP)** node (`Dispatch Approved Email`).
9. For WhatsApp and Instagram: Add HTTP request nodes to update statuses (`Sending`, `Sent`, `Failed`) and dispatch messages via external worker URLs, including fallback handling for manual reviews via `Arrange / Log Manual Review`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Complete Lead Generation Automation System | Primary workflow title and architectural definition |