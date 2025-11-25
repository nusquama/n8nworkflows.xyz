Monitor & Respond to Google Maps Reviews using AI, Apify, Slack, and Sheets

https://n8nworkflows.xyz/workflows/monitor---respond-to-google-maps-reviews-using-ai--apify--slack--and-sheets-11053


# Monitor & Respond to Google Maps Reviews using AI, Apify, Slack, and Sheets

### 1. Workflow Overview

This n8n workflow automates the monitoring and response process for Google Maps reviews of a physical store using AI, Apify, Slack, and Google Sheets. It is designed to run on a daily schedule, retrieve the latest reviews through a web scraper, filter out duplicates using Google Sheets, analyze the reviews with an AI agent to generate summaries and reply drafts, alert the team via Slack based on review sentiment, and save all processed data back into Google Sheets for record-keeping.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Configuration**: Initiates the workflow every 24 hours and sets configuration parameters such as store URL and Google Sheet details.
- **1.2 Data Retrieval**: Scrapes Google Maps reviews using Apify and fetches existing review IDs from Google Sheets to detect duplicates.
- **1.3 Filtering and Deduplication**: Compares newly scraped reviews against existing ones to pass only new reviews forward.
- **1.4 AI Analysis**: Uses an AI agent to generate both a concise summary and a tailored reply draft based on star ratings.
- **1.5 Post-Processing and Saving**: Parses AI output, combines it with original review data, and appends results to Google Sheets.
- **1.6 Alerting via Slack**: Sends notifications to appropriate Slack channels depending on the positivity or negativity of the review.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Configuration

- **Overview:**  
  This block triggers the workflow every 24 hours and sets essential configuration parameters like the Google Maps URL, Google Sheet ID, store name, and manager name.

- **Nodes Involved:**  
  - Every 24 Hours (Schedule Trigger)  
  - CONFIG (Edit Here) (Set Node)

- **Node Details:**

  - **Every 24 Hours**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow on a fixed interval (every 24 hours).  
    - *Configuration:* Interval set to 24 hours.  
    - *Input:* None (trigger)  
    - *Output:* Triggers CONFIG node.  
    - *Potential Failures:* Scheduler misconfiguration, system downtime.

  - **CONFIG (Edit Here)**  
    - *Type:* Set Node  
    - *Role:* Holds all user-editable configuration parameters for easy setup.  
    - *Configuration:* Stores MAPS_URL (Google Maps store URL), SHOP_NAME, MANAGER_NAME, SHEET_ID, SHEET_NAME.  
    - *Key Expressions:* Parameters are injected into downstream nodes using expressions like `={{ $json.MAPS_URL }}`.  
    - *Input:* Trigger from scheduler.  
    - *Output:* Feeds data retrieval nodes.  
    - *Potential Failures:* Incorrect or missing config values will break downstream nodes. Requires manual editing before first run.  

---

#### 2.2 Data Retrieval

- **Overview:**  
  This block scrapes the latest Google Maps reviews via Apify and retrieves existing review IDs from Google Sheets for duplicate checking.

- **Nodes Involved:**  
  - Get Existing IDs (Google Sheets)  
  - Run an Actor and get dataset (Apify)

- **Node Details:**

  - **Get Existing IDs**  
    - *Type:* Google Sheets  
    - *Role:* Reads existing reviews stored in the Google Sheet to obtain review IDs for duplicate filtering.  
    - *Configuration:* Reads from the sheet named "Reviews" (gid=0) in the document id provided by CONFIG node.  
    - *Input:* Receives config from CONFIG node.  
    - *Output:* Provides existing review data for filtering duplicates.  
    - *Potential Failures:* Google Sheets API errors, incorrect sheet ID, permission issues.

  - **Run an Actor and get dataset**  
    - *Type:* Apify Node  
    - *Role:* Runs the Google Maps Reviews Scraper Actor on Apify to fetch the latest reviews.  
    - *Configuration:*  
      - Actor ID: `Xb8osYTtOjlsgI6k9` (Google Maps Reviews Scraper)  
      - Input JSON includes:  
        - `startUrls` with URL from CONFIG `MAPS_URL`  
        - `maxReviews`: 10 (max number of reviews to fetch)  
        - `language`: "en"  
        - `reviewsSort`: "newest"  
    - *Input:* Receives config for URL from CONFIG node.  
    - *Output:* Outputs an array of latest reviews.  
    - *Potential Failures:* Apify API errors, actor not available, network issues, invalid URL.

---

#### 2.3 Filtering and Deduplication

- **Overview:**  
  This block compares newly scraped reviews against existing ones in the Google Sheet to ensure only new reviews are processed further.

- **Nodes Involved:**  
  - Filter Duplicates (Merge Node)

- **Node Details:**

  - **Filter Duplicates**  
    - *Type:* Merge Node  
    - *Role:* Combines Apify data and existing Google Sheets data to pass only records that do not have matching review IDs.  
    - *Configuration:*  
      - Mode: Combine  
      - Join Mode: Keep Non-Matches (keeps only reviews with reviewId not found in existing data)  
      - Merge by fields: `reviewerId` (Apify data) and `reviewId` (Google Sheets data)  
    - *Input:* Receives Apify scraped reviews and existing Google Sheets reviews.  
    - *Output:* Passes filtered unique reviews to AI Agent.  
    - *Potential Failures:* Expression errors if review IDs are missing or mismatched, data inconsistencies.

---

#### 2.4 AI Analysis

- **Overview:**  
  This block sends each new review through an AI agent that generates a short summary and a reply draft based on the review text and star rating.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - Code in JavaScript (Code Node)

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent Node  
    - *Role:* Processes each review to generate two outputs: a summary (for Slack) and a reply draft (for Google Sheets).  
    - *Configuration:*  
      - Text Input: template including Review Text, Stars, Reviewer Name  
      - System Message instructs the AI on the persona (customer support manager), response format, tone, and rules for reply drafting based on star rating.  
      - Output Format explicitly structured as:  
        ```
        [Summary]
        (summary text)
        
        [Reply Draft]
        (reply text)
        ```  
    - *Input:* Receives filtered new reviews from Filter Duplicates.  
    - *Output:* Raw AI text output containing both summary and reply draft.  
    - *Potential Failures:* API rate limits, malformed input, AI model errors.

  - **Code in JavaScript**  
    - *Type:* Function Node  
    - *Role:* Parses AI agent output to separate the summary and reply draft and combines them with original review data.  
    - *Configuration:*  
      - Uses regex to extract `[Summary]` and `[Reply Draft]` sections.  
      - Merges AI output with original review fields (reviewId, stars, text, etc.).  
    - *Input:* Receives AI Agent output and original review data from Filter Duplicates.  
    - *Output:* Structured JSON objects ready for saving.  
    - *Potential Failures:* Parsing errors if AI output deviates from expected format.

---

#### 2.5 Post-Processing and Saving

- **Overview:**  
  This block appends the enriched reviews with AI-generated data into the Google Sheet and triggers Slack alerts based on the star rating.

- **Nodes Involved:**  
  - Save to Google Sheets (Google Sheets)  
  - If Rating < 4 (If Node)  
  - Slack (Alert) (Slack Node)  
  - Slack (Alert)1 (Slack Node)

- **Node Details:**

  - **Save to Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Appends new reviews with AI summaries and replies into the configured Google Sheet.  
    - *Configuration:*  
      - Columns mapped explicitly: text, stars, output, ai_reply, reviewId, reviewUrl, ai_summary, publishedAt, reviewerName, publishedAt date.  
      - Sheet ID and name dynamically pulled from CONFIG node.  
      - Operation: Append.  
    - *Input:* Receives processed review data from Code in JavaScript node.  
    - *Output:* Passes data for rating evaluation.  
    - *Potential Failures:* Sheet permission issues, wrong sheet ID, missing columns.

  - **If Rating < 4**  
    - *Type:* If Node  
    - *Role:* Routes the flow based on the star rating to send appropriate Slack alerts.  
    - *Configuration:* Condition tests if `reviewId` (likely a misconfigured field; should be star rating) is less than 4. (Note: This looks like an error; should check stars, not reviewId.)  
    - *Input:* From Save to Google Sheets.  
    - *Output:*  
      - True branch: Slack (Alert) node for negative reviews.  
      - False branch: Slack (Alert)1 node for positive reviews.  
    - *Potential Failures:* Logical error due to incorrect field comparison, causing wrong alert routing.

  - **Slack (Alert)**  
    - *Type:* Slack Node  
    - *Role:* Sends alert for negative reviews to a specified Slack channel.  
    - *Configuration:*  
      - Text: "🚨 Negative review detected"  
      - Channel ID: Configured for negative alerts (e.g., customer support channel)  
      - Authentication: OAuth2  
      - Includes link to the workflow for context.  
    - *Input:* From If Rating < 4 (true branch).  
    - *Output:* None.  
    - *Potential Failures:* Slack API permission issues, invalid channel ID.

  - **Slack (Alert)1**  
    - *Type:* Slack Node  
    - *Role:* Sends alert for positive reviews to a specified Slack channel.  
    - *Configuration:*  
      - Text: "🚨 Positive review received!"  
      - Channel ID: Configured for positive alerts (e.g., general or wins channel)  
      - Authentication: OAuth2  
      - Includes link to workflow.  
    - *Input:* From If Rating < 4 (false branch).  
    - *Output:* None.  
    - *Potential Failures:* Slack API permission issues, invalid channel ID.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                                  | Input Node(s)                   | Output Node(s)              | Sticky Note                                                                                                                            |
|-------------------------------|------------------------|-------------------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Every 24 Hours                | Schedule Trigger       | Triggers workflow every 24 hours                 | None                          | CONFIG (Edit Here)           | ## How it works<br>1. **Schedule:** Runs every 24 hours (customizable) to fetch the latest data.                                        |
| CONFIG (Edit Here)            | Set                    | Holds user configuration parameters              | Every 24 Hours                | Get Existing IDs, Run an Actor and get dataset | Settings: Enter your Store URL and Sheet ID here                                                                                       |
| Get Existing IDs              | Google Sheets          | Reads existing reviews from Google Sheets        | CONFIG (Edit Here)            | Filter Duplicates            | For duplicate check                                                                                                                    |
| Run an Actor and get dataset  | Apify                  | Scrapes latest Google Maps reviews                | CONFIG (Edit Here)            | Filter Duplicates            |                                                                                                                                         |
| Filter Duplicates             | Merge                  | Filters out duplicate reviews                      | Get Existing IDs, Run an Actor and get dataset | AI Agent                   | Pass only new reviews                                                                                                                  |
| AI Agent                     | LangChain Agent        | Generates summary and reply draft from review text | Filter Duplicates             | Code in JavaScript           |                                                                                                                                         |
| Code in JavaScript            | Code                   | Parses AI output and merges with original review | AI Agent                     | Save to Google Sheets        |                                                                                                                                         |
| Save to Google Sheets         | Google Sheets          | Appends processed reviews to Google Sheet         | Code in JavaScript            | If Rating < 4                |                                                                                                                                         |
| If Rating < 4                 | If                     | Routes flow based on star rating                   | Save to Google Sheets         | Slack (Alert), Slack (Alert)1 |                                                                                                                                         |
| Slack (Alert)                 | Slack                  | Sends alert for negative reviews                   | If Rating < 4 (true branch)   | None                        |                                                                                                                                         |
| Slack (Alert)1                | Slack                  | Sends alert for positive reviews                   | If Rating < 4 (false branch)  | None                        |                                                                                                                                         |
| Sticky Note                  | Sticky Note            | Documentation on workflow overview and requirements | None                         | None                        | ## How it works<br>...includes scheduling, scraping, filtering, AI analysis, alerts, saving, and requirements.                         |
| Sticky Note1                 | Sticky Note            | Setup instructions                                 | None                         | None                        | ## How to set up<br>Includes Google Sheets headers, credentials, config edits, and Slack channel setup.                                |
| Sticky Note2                 | Sticky Note            | Customization tips                                 | None                         | None                        | ## How to customize<br>Change AI persona, alert thresholds, and multi-store support.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**
   - Type: Schedule Trigger  
   - Set interval to every 24 hours.

2. **Create a Set node named "CONFIG (Edit Here)":**
   - Add string parameters:  
     - `MAPS_URL`: Full Google Maps store URL (e.g., `https://www.google.com/maps/place/YOUR_STORE_ID`)  
     - `SHOP_NAME`: Your store’s name  
     - `MANAGER_NAME`: Manager’s name  
     - `SHEET_ID`: Google Sheet ID (found in your sheet URL)  
     - `SHEET_NAME`: Sheet tab name, e.g., "Reviews"

3. **Connect Schedule Trigger to CONFIG node.**

4. **Create a Google Sheets node named "Get Existing IDs":**
   - Operation: Read (default)  
   - Document ID: Use expression to get from CONFIG: `={{ $json.SHEET_ID }}`  
   - Sheet Name: "Reviews" (gid=0)  
   - No filters (fetch all rows)  
   - Connect CONFIG node to this node.

5. **Create an Apify node named "Run an Actor and get dataset":**
   - Actor ID: `Xb8osYTtOjlsgI6k9` (Google Maps Reviews Scraper)  
   - Operation: Run actor and get dataset  
   - Custom Body (JSON input):  
     ```json
     {
       "startUrls": [{"url": "={{ $('CONFIG (Edit Here)').item.json.MAPS_URL }}"}],
       "maxReviews": 10,
       "language": "en",
       "reviewsSort": "newest"
     }
     ```  
   - Connect CONFIG node to this node.

6. **Create a Merge node named "Filter Duplicates":**
   - Mode: Combine  
   - Join Mode: Keep Non-Matches  
   - Merge By Fields:  
     - Field 1: `reviewerId` (from Apify data)  
     - Field 2: `reviewId` (from Google Sheets data)  
   - Connect "Get Existing IDs" to input 2 and "Run an Actor and get dataset" to input 1.

7. **Create a LangChain Agent node named "AI Agent":**
   - Text input template:  
     ```
     Review Text: {{ $json.text }}
     Stars: {{ $json.stars }}
     Reviewer Name: {{ $json.reviewerName }}
     ```
   - System Message:  
     ```
     You are an excellent customer support manager for a physical store.
     Based on the "Google Maps Review" and "Star Rating (1-5)" provided by the user, create the following two items:

     ### 1. Summary (For Slack Notification)
     Summarize the main point of the review in under 30 characters so the store manager can grasp it immediately via notification.

     ### 2. Reply Draft (For Spreadsheet)
     Create an appropriate reply message from the store.
     Strictly follow these rules:
     - 4-5 Stars: Thank them for the high rating and warmly say you look forward to their next visit.
     - 1-3 Stars: Sincerely apologize for the inconvenience, thank them for the valuable feedback, and express a polite attitude towards improvement.
     - Tone: Polite (formal) but sincere and warm.
     - No signature needed.

     ### Output Format
     Do not include any preamble or greetings. Output only the following format:

     [Summary]
     (Summary text here)

     [Reply Draft]
     (Reply draft here)
     ```
   - Connect "Filter Duplicates" node to this node.

8. **Create a Code node named "Code in JavaScript":**
   - JavaScript code to parse AI output and merge with original data:  
     ```javascript
     const originalItems = $('Filter Duplicates').all();
     const aiItems = $input.all();

     return aiItems.map((item, index) => {
       const aiText = item.json.output || item.json.text || "";

       const summaryMatch = aiText.match(/\[Summary\]\s*([\s\S]*?)\s*(?=\[Reply Draft\]|$)/);
       const replyMatch = aiText.match(/\[Reply Draft\]\s*([\s\S]*)/);

       const summary = summaryMatch ? summaryMatch[1].trim() : "";
       const reply = replyMatch ? replyMatch[1].trim() : "";

       const originalData = originalItems[index] ? originalItems[index].json : {};

       return {
         json: {
           ...originalData,
           ai_summary: summary,
           ai_reply: reply,
           output: aiText
         }
       };
     });
     ```
   - Connect "AI Agent" node to this node.

9. **Create a Google Sheets node named "Save to Google Sheets":**
   - Operation: Append  
   - Document ID: `={{ $('CONFIG (Edit Here)').item.json.SHEET_ID }}`  
   - Sheet Name: "Reviews" (gid=0)  
   - Columns mapped as:  
     - `text` ← `={{ $json.text }}`  
     - `stars` ← `={{ $json.stars }}`  
     - `output` ← `={{ $json.output }}`  
     - `ai_reply` ← `={{ $json.ai_reply }}`  
     - `reviewId` ← `={{ $json.reviewerId }}`  
     - `reviewUrl` ← `={{ $json.reviewerUrl }}`  
     - `ai_summary` ← `={{ $json.ai_summary }}`  
     - `publishedAt` ← `={{ $json.publishAt }}`  
     - `reviewerName` ← `={{ $json.name }}`  
     - `publishedAt date` ← `={{ $json.publishedAtDate }}`  
   - Connect "Code in JavaScript" node to this node.

10. **Create an If node named "If Rating < 4":**
    - Condition:  
      - Number comparison  
      - Value 1: Use star rating field (correct this from `reviewId` to `stars`)  
      - Operator: Less than  
      - Value 2: 4  
    - Connect "Save to Google Sheets" node to this node.

11. **Create two Slack nodes:**

    - **Slack (Alert)** for negative reviews:  
      - Text: "🚨 Negative review detected"  
      - Channel: Select your negative alert Slack channel  
      - Authentication: OAuth2  
      - Connect If node "true" branch here.

    - **Slack (Alert)1** for positive reviews:  
      - Text: "🚨 Positive review received!"  
      - Channel: Select your positive alert Slack channel  
      - Authentication: OAuth2  
      - Connect If node "false" branch here.

12. **Verify all connections and credentials:**
    - Google Sheets credentials with read/write access.  
    - Apify account credentials with access to the Google Maps Reviews Scraper actor.  
    - Slack OAuth2 credentials with permissions to post messages in the selected channels.  
    - OpenRouter or OpenAI API key configured and connected for LangChain Agent.

13. **Test the workflow:**
    - Run manually to verify data flows through all nodes correctly.  
    - Check Slack channels for notifications.  
    - Validate Google Sheet for appended data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| **Workflow Overview:** Runs every 24 hours, scrapes Google Maps reviews, filters duplicates, uses AI to summarize and draft replies, sends Slack alerts based on review sentiment, and saves enriched data to Google Sheets. Requires Apify, Google Sheets API, Slack OAuth2, and OpenRouter/OpenAI API.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | See Sticky Note at workflow start                                                                 |
| **Setup instructions:** Create Google Sheet with headers: `reviewId`, `publishedAt`, `reviewerName`, `stars`, `text`, `ai_summary`, `ai_reply`, `reviewUrl`, `output`, `publishedAt date`. Configure all credentials and edit CONFIG node with your store details and sheet ID. Select proper Slack channels for alerts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | See Sticky Note1                                                                                   |
| **Customization tips:** Change AI persona by editing AI Agent system message. Adjust alert thresholds by modifying the "If Rating < 4" node condition. Support multiple stores by looping over URLs externally.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | See Sticky Note2                                                                                   |
| **Important:** The "If Rating < 4" node currently compares the `reviewId` field instead of the `stars` rating, which is a probable misconfiguration and should be corrected to ensure proper alert routing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Manual correction recommended                                                                       |
| **Apify Actor used:** Google Maps Reviews Scraper by Apify (ID: `Xb8osYTtOjlsgI6k9`). Documentation available at Apify Console URL: https://console.apify.com/actors/Xb8osYTtOjlsgI6k9/input                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Apify actor details                                                                                |
| **Slack Notifications:** Includes a link back to the workflow for context in alerts, which helps team members quickly access the automation details when alerts trigger.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Slack node configuration                                                                          |

---

**Disclaimer:** The provided documentation strictly reflects the given n8n workflow JSON. All data processed is compliant with applicable content policies and legal standards.