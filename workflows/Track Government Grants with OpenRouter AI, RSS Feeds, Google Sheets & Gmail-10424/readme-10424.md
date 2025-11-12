Track Government Grants with OpenRouter AI, RSS Feeds, Google Sheets & Gmail

https://n8nworkflows.xyz/workflows/track-government-grants-with-openrouter-ai--rss-feeds--google-sheets---gmail-10424


# Track Government Grants with OpenRouter AI, RSS Feeds, Google Sheets & Gmail

### 1. Workflow Overview

This workflow is designed to **automatically track government grants** by fetching updates from government RSS feeds, processing and classifying them with AI, storing structured data in Google Sheets, and sending summary notifications by email. It is aimed at users or organizations who want to monitor public grant opportunities efficiently without manual checking.

The logic is organized into the following functional blocks:

- **1.1 Scheduled Trigger & RSS Feed Retrieval**  
  Periodically triggers the workflow and fetches new grant-related updates from a government RSS feed.

- **1.2 Batch Processing & AI Classification**  
  Splits the feed items into batches to manage load and API rate limits; classifies each item into categories using AI.

- **1.3 AI Extraction & Structured Parsing**  
  Uses an AI agent with OpenRouter chat model to extract detailed grant information into a consistent JSON schema.

- **1.4 Data Upsert into Google Sheets**  
  Appends or updates structured grant data into a Google Sheets spreadsheet keyed by grant title.

- **1.5 Aggregation & Email Notification**  
  Aggregates all new grants for the current run and generates an HTML email summary sent via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & RSS Feed Retrieval

- **Overview:**  
  This block initiates the workflow on a regular schedule and fetches the latest grant announcements from a specified RSS feed.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RSS Feed Reader  
  - Notes: Schedule, RSS

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Trigger node to start workflow at defined intervals.  
    - Config: Runs on a recurring schedule (default hourly interval).  
    - Inputs: None (start node).  
    - Outputs: Triggers RSS Feed Reader.  
    - Edge Cases: Misconfigured schedule may cause missed or excessive runs.

  - **RSS Feed Reader**  
    - Type: Reads RSS feed content.  
    - Config: URL set to `https://www.mhlw.go.jp/stf/news.rdf` (Ministry of Health, Labour and Welfare Japan).  
    - Inputs: Trigger from Schedule node.  
    - Outputs: List of RSS items to batch processor.  
    - Edge Cases: Feed unavailability, malformed RSS, network timeout.

  - **Note: Schedule & Note: RSS**  
    - Visual sticky notes explaining the schedule and RSS feed purpose.

---

#### 1.2 Batch Processing & AI Classification

- **Overview:**  
  Processes feed items in smaller batches to improve stability and respect API limits; classifies each item’s content into predefined categories.

- **Nodes Involved:**  
  - Loop Over Items (Split in Batches)  
  - Text Classifier (LangChain)  
  - Notes: Loop, Classifier

- **Node Details:**  
  - **Loop Over Items**  
    - Type: Splits input array into batches of 10 items.  
    - Config: Batch size = 10; no additional options.  
    - Inputs: Feed items array from RSS Feed Reader.  
    - Outputs: Each batch sent to Google Sheets node and Text Classifier node.  
    - Edge Cases: Large feeds may increase runtime; batch size may be adjusted for API limits.

  - **Text Classifier**  
    - Type: LangChain AI classifier node.  
    - Config: Classifies input text (concatenated title + summary) into three categories: "Grant/Subsidy", "Labor-related", "Other". Uses fallback category "Other".  
    - Inputs: Individual feed items from Loop Over Items.  
    - Outputs: Categorized items for further AI extraction.  
    - Edge Cases: Misclassification if input text is ambiguous; fallback prevents failure.

  - **Note: Loop & Classifier**  
    - Explain batching for stability and AI classification for routing/filtering.

---

#### 1.3 AI Extraction & Structured Parsing

- **Overview:**  
  Extracts detailed grant information into a consistent JSON format using an AI agent powered by OpenRouter chat model, with output parsing to ensure valid JSON.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent node)  
  - OpenRouter Chat Model (Language Model)  
  - Structured Output Parser (LangChain Output Parser)  
  - Notes: Agent, Parser

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain agent configured to transform RSS item fields into structured JSON.  
    - Config: System message instructs to extract title, summary, deadline, amount, target, URL, and published date. Outputs only valid JSON. Uses input text from feed item JSON.  
    - Inputs: Output from Text Classifier (categorized feed item).  
    - Outputs: Raw AI JSON output parsed by Structured Output Parser.  
    - Edge Cases: AI model may return malformed JSON (mitigated by output parser); network or API errors with OpenRouter.

  - **OpenRouter Chat Model (for Agent)**  
    - Type: Language Model node providing AI completion via OpenRouter API.  
    - Config: Uses stored OpenRouter API credentials.  
    - Inputs: Requests from AI Agent node.  
    - Outputs: AI-generated text for AI Agent.  
    - Edge Cases: API rate limits, authentication errors, latency.

  - **Structured Output Parser**  
    - Type: Output parser node that validates and structures AI output according to a JSON schema example.  
    - Config: JSON schema example includes all required fields with sample data to ensure consistency.  
    - Inputs: AI Agent’s output.  
    - Outputs: Structured JSON ready for storage/upsert.  
    - Edge Cases: Parser failure if AI output is invalid JSON or schema does not match.

  - **Note: Agent & Parser**  
    - Explain AI extraction purpose and importance of schema validation to prevent malformed data.

---

#### 1.4 Data Upsert into Google Sheets

- **Overview:**  
  Inserts or updates grant data rows in a Google Sheets document keyed by grant title, enabling a live database of tracked grants.

- **Nodes Involved:**  
  - Google Sheets  
  - Notes: Sheets

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets node.  
    - Config: Operation set to "appendOrUpdate" using "タイトル" (Title) as matching column. Columns mapped to five generic fields (likely mapped to title, summary, deadline, amount, etc.). Target spreadsheet and sheet specified by Spreadsheet ID and gid=0 (sheet "Grants").  
    - Inputs: Individual feed items processed in batches from Loop Over Items node.  
    - Outputs: Aggregated data for downstream steps.  
    - Credentials: Uses OAuth2 credentials for Google Sheets.  
    - Edge Cases: API quota exceeded, authentication errors, spreadsheet inaccessible, column mismatch.

  - **Note: Sheets**  
    - Documentation about upsert behavior and advice to customize column headers according to the user’s sheet.

---

#### 1.5 Aggregation & Email Notification

- **Overview:**  
  Aggregates all newly processed grants into a single payload, generates an HTML table summary, and emails it to the configured recipients.

- **Nodes Involved:**  
  - Aggregate  
  - Code in JavaScript  
  - Gmail  
  - Notes: Aggregate, HTML, Gmail

- **Node Details:**  
  - **Aggregate**  
    - Type: Aggregates all item data into a single combined output.  
    - Config: Uses "aggregateAllItemData" to collapse items into one array.  
    - Inputs: Outputs from Google Sheets (processed items).  
    - Outputs: Single aggregated payload for HTML generation.  
    - Edge Cases: Empty input results in empty email content; large payloads might require timeout considerations.

  - **Code in JavaScript**  
    - Type: Function node to generate an HTML email body.  
    - Config: Builds a styled HTML table with columns Title, Summary, URL, Deadline, Amount, Target, Published. Uses inline CSS styling. Iterates over aggregated items to fill rows. Returns HTML string as `html_body`.  
    - Inputs: Aggregated grant data.  
    - Outputs: HTML string for email body.  
    - Edge Cases: Missing fields default to empty string; malformed data could break table but handled gracefully.

  - **Gmail**  
    - Type: Gmail node to send email.  
    - Config: Sends email to `you@example.com` (placeholder), subject "New Grants Detected", using HTML body from previous node.  
    - Credentials: Gmail OAuth2 credentials required.  
    - Inputs: HTML email content from Code node.  
    - Outputs: Email sent confirmation.  
    - Edge Cases: Invalid email address, Gmail API quota, authentication failure.

  - **Note: Aggregate, HTML, Gmail**  
    - Explain aggregation, HTML email formatting, and email sending with customization notes.

---

### 3. Summary Table

| Node Name                     | Node Type                                   | Functional Role                       | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                   |
|-------------------------------|---------------------------------------------|-------------------------------------|-----------------------------|---------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger               | n8n-nodes-base.scheduleTrigger              | Initiate workflow on schedule       | None                        | RSS Feed Reader            | $23F0 Schedule; $2022 Entry point of the workflow; $2022 Set how often this flow runs          |
| Note: Schedule                | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | $23F0 Schedule; $2022 Entry point of the workflow; $2022 Set how often this flow runs          |
| RSS Feed Reader               | n8n-nodes-base.rssFeedRead                   | Fetch RSS grant updates             | Schedule Trigger            | Loop Over Items            | 📱 RSS Feed; $2022 Fetch grant-related updates from MHLW RSS; $2022 You may add more RSS feeds |
| Note: RSS                    | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 📱 RSS Feed; $2022 Fetch grant-related updates from MHLW RSS; $2022 You may add more RSS feeds |
| Loop Over Items              | n8n-nodes-base.splitInBatches                | Batch processing of feed items     | RSS Feed Reader             | Google Sheets, Text Classifier | 🔁 Batching; $2022 Process feed items in batches; $2022 Helps with stability and API rate limits |
| Note: Loop                  | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🔁 Batching; $2022 Process feed items in batches; $2022 Helps with stability and API rate limits |
| Text Classifier             | @n8n/n8n-nodes-langchain.textClassifier      | Categorize feed items with AI      | Loop Over Items             | AI Agent                  | 🏷️ AI Classification; $2022 Categorize each item by title/summary; $2022 Used for filtering/routing |
| Note: Classifier            | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🏷️ AI Classification; $2022 Categorize each item by title/summary; $2022 Used for filtering/routing |
| AI Agent                   | @n8n/n8n-nodes-langchain.agent                | Extract structured grant info      | Text Classifier             | Structured Output Parser   | 🧖 AI Extraction; $2022 Convert each feed item into consistent JSON structure; $2022 Extract deadline/amount/target/URL/published date |
| Note: Agent                | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🧖 AI Extraction; $2022 Convert each feed item into consistent JSON structure; $2022 Extract deadline/amount/target/URL/published date |
| OpenRouter Chat Model (for Agent) | @n8n/n8n-nodes-langchain.lmChatOpenRouter | AI language model for agent        | AI Agent                    | AI Agent                  |                                                                                               |
| Structured Output Parser     | @n8n/n8n-nodes-langchain.outputParserStructured | Validate and structure AI output  | AI Agent                   | AI Agent (ai_outputParser) | 🧩 Output Parser; $2022 Provides example schema to keep keys consistent; $2022 Helps prevent malformed JSON |
| Note: Parser               | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🧩 Output Parser; $2022 Provides example schema to keep keys consistent; $2022 Helps prevent malformed JSON |
| Google Sheets              | n8n-nodes-base.googleSheets                   | Store/upsert grant data             | Loop Over Items             | Aggregate                 | 🗃️ Google Sheets Upsert; $2022 Append or update rows keyed by Title; $2022 Adjust column headers to your sheet |
| Note: Sheets               | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🗃️ Google Sheets Upsert; $2022 Append or update rows keyed by Title; $2022 Adjust column headers to your sheet |
| Aggregate                 | n8n-nodes-base.aggregate                       | Combine processed items for report | Google Sheets               | Code in JavaScript        | 🧮 Aggregate; $2022 Combine newly processed items for reporting; $2022 Pass a single payload downstream |
| Note: Aggregate           | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🧮 Aggregate; $2022 Combine newly processed items for reporting; $2022 Pass a single payload downstream |
| Code in JavaScript        | n8n-nodes-base.code                            | Generate HTML summary email body   | Aggregate                   | Gmail                     | 🎱 HTML Builder; $2022 Generate an HTML table for the email body; $2022 Customize columns and styles as needed |
| Note: HTML                | n8n-nodes-base.stickyNote                    | Documentation                       | None                        | None                      | 🎱 HTML Builder; $2022 Generate an HTML table for the email body; $2022 Customize columns and styles as needed |
| Gmail                     | n8n-nodes-base.gmail                           | Send email notification             | Code in JavaScript          | None                      | ⚙️ Gmail; $2022 Send the HTML summary to your recipients; $2022 Replace 'sendTo' and subject per environment |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to desired frequency (e.g., hourly, daily).  
   - No inputs.  

2. **Add RSS Feed Reader node:**  
   - Connect from Schedule Trigger.  
   - Set RSS feed URL to `https://www.mhlw.go.jp/stf/news.rdf`.  
   - Leave options default.

3. **Add Split In Batches node ("Loop Over Items"):**  
   - Connect from RSS Feed Reader.  
   - Configure batch size = 10.

4. **Add Google Sheets node:**  
   - Connect one output of "Loop Over Items" to Google Sheets.  
   - Configure operation as "appendOrUpdate" with matching column "タイトル" (Title).  
   - Map columns with your sheet headers (e.g., Title, Summary, Deadline, Amount, Target, URL, Published).  
   - Set Spreadsheet ID and Sheet GID to your target Google Sheet.  
   - Add and configure Google Sheets OAuth2 credentials.

5. **Add Text Classifier node:**  
   - Connect second output of "Loop Over Items" to Text Classifier.  
   - Use input expression: `{{$json["title"] + " " + ($json["contentSnippet"] || "")}}`.  
   - Define categories:  
     - "Grant/Subsidy" (Grants, subsidies, public calls)  
     - "Labor-related" (Labor safety, regulations, press, stats)  
     - "Other" (fallback category).  
   - Set fallback category to "Other".

6. **Add AI Agent node:**  
   - Connect output of Text Classifier to AI Agent.  
   - Set system message prompt instructing extraction of grant details (title, summary, deadline, amount, target, url, published_date) from RSS fields into valid JSON only.  
   - Set input text to entire feed item JSON.

7. **Add OpenRouter Chat Model node:**  
   - Connect to AI Agent as language model.  
   - Configure with OpenRouter API credentials.

8. **Add Structured Output Parser node:**  
   - Connect AI Agent output to this node’s parser input.  
   - Provide example JSON schema for grants with fields: title, summary, deadline, amount, target, url, published_date.

9. **Connect Structured Output Parser output back to AI Agent output parser input** (as per n8n LangChain integration).

10. **Connect AI Agent output to Loop Over Items’ next step or directly to Google Sheets if feeding parsed structured data.**

11. **Add Aggregate node:**  
    - Connect output of Google Sheets node.  
    - Configure to aggregate all item data into a single array.

12. **Add Code node:**  
    - Connect output of Aggregate.  
    - Paste JavaScript code to generate an HTML table with columns: Title, Summary, URL, Deadline, Amount, Target, Published.  
    - This code iterates over aggregated items and builds an HTML email body.

13. **Add Gmail node:**  
    - Connect output of Code node.  
    - Set recipient email address (replace `you@example.com` with your own).  
    - Use the HTML from Code node as the email body.  
    - Set email subject as "New Grants Detected".  
    - Configure Gmail OAuth2 credentials.

14. **Add sticky notes to document each major section for clarity (optional).**

15. **Activate the workflow and test by running manually or waiting for scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| This workflow uses OpenRouter API for AI processing; ensure your OpenRouter account and API key are valid. | OpenRouter: https://openrouter.ai/                                 |
| Google Sheets columns must match the column headers in your spreadsheet for correct upsert operation.      | Customize in Google Sheets node parameters                           |
| Replace email address in Gmail node parameter to your actual recipient address before production use.      | Gmail node configuration                                            |
| Batch size in Split In Batches node can be adjusted according to API limits and performance requirements.  | To optimize API usage and avoid rate limiting                        |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.