Japanese-to-English Review Sentiment Analysis with GPT and Telegram Alerts

https://n8nworkflows.xyz/workflows/japanese-to-english-review-sentiment-analysis-with-gpt-and-telegram-alerts-11245


# Japanese-to-English Review Sentiment Analysis with GPT and Telegram Alerts

### 1. Workflow Overview

This workflow automates the analysis of customer product reviews written in Japanese by leveraging OpenAI GPT and integrating Google Sheets and Telegram for data management and alerting. It is designed for e-commerce, customer support, and marketing teams to monitor reviews, translate them, conduct sentiment and importance analysis, categorize feedback, and prioritize urgent issues via Telegram notifications.

**Logical Blocks:**

- **1.1 Trigger & Data Fetch**  
  Periodic scheduling and retrieval of review data from Google Sheets.

- **1.2 Filter & Format**  
  Filtering out already processed reviews and preparing data for AI analysis.

- **1.3 AI Processing**  
  Using OpenAI GPT to translate reviews, evaluate sentiment and importance, classify categories, and extract key phrases.

- **1.4 Save & Notify**  
  Writing AI results back to Google Sheets and sending Telegram alerts based on priority and sentiment.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Data Fetch

- **Overview:**  
  This block initiates the workflow on a schedule and fetches review data from a Google Sheets document for further processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Review Data

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution at defined intervals (default every minute).  
    - Configuration: Interval left at default (every minute).  
    - Inputs: None  
    - Outputs: Triggers "Get Review Data"  
    - Edge Cases: Misconfigured schedule could cause missing or excessive runs.

  - **Get Review Data**  
    - Type: Google Sheets node  
    - Role: Retrieves all rows from the specified Google Sheet (Sheet1).  
    - Configuration: Reads from the sheet with documentId and sheetName set to "gid=0" (Sheet1).  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Passes review data array to "Filter Unprocessed"  
    - Edge Cases: Authentication or API quota errors; empty or malformed sheets.

---

#### 1.2 Filter & Format

- **Overview:**  
  Filters reviews to only those not yet processed (empty ProcessStatus) and prepares the data structure for AI input.

- **Nodes Involved:**  
  - Filter Unprocessed  
  - Format Input Data

- **Node Details:**

  - **Filter Unprocessed**  
    - Type: If node  
    - Role: Filters incoming data to pass on only reviews where ProcessStatus is empty (unprocessed).  
    - Configuration: Condition equals "" on ProcessStatus field.  
    - Inputs: Data from "Get Review Data"  
    - Outputs: Passes filtered reviews to "Format Input Data"  
    - Edge Cases: Empty or missing ProcessStatus fields; case sensitivity handled strictly.

  - **Format Input Data**  
    - Type: Set node  
    - Role: Sets or reformats data fields for consistent input to the AI agent. Does not specify explicit parameters, likely passes all input fields forward.  
    - Inputs: Filtered reviews  
    - Outputs: Data structured for AI agent consumption  
    - Edge Cases: Unexpected or missing fields in input data.

---

#### 1.3 AI Processing

- **Overview:**  
  Uses OpenAI GPT to translate Japanese reviews to English, analyze sentiment and importance, classify categories, and extract key phrases using a defined prompt. Response is parsed and merged with original data.

- **Nodes Involved:**  
  - AI Agent - Review Analysis  
  - OpenAI Chat Model (internal to AI Agent)  
  - Parse AI Response

- **Node Details:**

  - **AI Agent - Review Analysis**  
    - Type: langchain.agent node  
    - Role: Sends formatted review text to OpenAI GPT with detailed instructions for translation, sentiment scoring, importance rating, category classification, and key phrase extraction.  
    - Configuration:  
      - System message: Defines agent as multilingual review analysis expert.  
      - Prompt: Detailed multi-task JSON output format including translation, sentiment, score, importance, category, key phrase.  
      - Temperature: 0.3 (via linked OpenAI Chat Model) for controlled output.  
    - Inputs: Formatted review data  
    - Outputs: Raw AI response text forwarded to "Parse AI Response"  
    - Edge Cases: API rate limits, unexpected AI output format, timeout, prompt misinterpretation.

  - **OpenAI Chat Model**  
    - Type: langchain.lmChatOpenAi  
    - Role: Underlying language model for AI Agent with temperature setting.  
    - Configuration: temperature=0.3 for balanced responses.  
    - Inputs: Prompt from AI Agent  
    - Outputs: Text response to AI Agent  

  - **Parse AI Response**  
    - Type: Code node (JavaScript)  
    - Role: Extracts JSON content from AI agent’s response, safely parses it, and merges parsed data with original input data.  
    - Configuration: Includes fallback defaults on parse failure to avoid workflow breakage.  
    - Inputs: AI agent output  
    - Outputs: Structured JSON with translated text, sentiment, sentiment score, importance, category, key phrase, plus metadata (ProcessStatus, ProcessedAt).  
    - Edge Cases: Malformed JSON, missing JSON block in response, parse exceptions handled gracefully with default values.

---

#### 1.4 Save & Notify

- **Overview:**  
  Updates Google Sheets with analyzed data and sends Telegram notifications. High-priority or negative sentiment reviews trigger urgent alerts; others receive standard notifications.

- **Nodes Involved:**  
  - Update Spreadsheet  
  - Check Importance & Sentiment  
  - Urgent Alert (Telegram)  
  - Normal Notification (Telegram)

- **Node Details:**

  - **Update Spreadsheet**  
    - Type: Google Sheets node  
    - Role: Updates the spreadsheet row for each review with analysis results (TranslatedText, Sentiment, SentimentScore, Importance, Category, KeyPhrase, ProcessStatus, ProcessedAt).  
    - Configuration:  
      - Operation: Update  
      - Matching column: ReviewID for row identification  
      - Column mapping: Auto-mapped from JSON fields  
      - Cell format: USER_ENTERED (to allow formulas or date formatting)  
    - Inputs: Parsed AI data  
    - Outputs: Passes updated data to "Check Importance & Sentiment"  
    - Edge Cases: Spreadsheet permission errors, matching failures, concurrent update conflicts.

  - **Check Importance & Sentiment**  
    - Type: If node  
    - Role: Checks if review is of High importance or SentimentScore is less than -0.5 (negative threshold) to decide notification type.  
    - Configuration: OR condition: Importance == "High" OR SentimentScore < -0.5  
    - Inputs: Updated review data  
    - Outputs: Routes to either "Urgent Alert (Telegram)" or "Normal Notification (Telegram)"  
    - Edge Cases: Missing or unexpected importance/sentiment values.

  - **Urgent Alert (Telegram)**  
    - Type: Telegram node  
    - Role: Sends an urgent message with detailed review analysis for high priority or negative reviews.  
    - Configuration:  
      - Text includes ReviewID, original review (OriginalText), translation, sentiment, importance, category, key phrase, and processed timestamp.  
      - Chat ID and Bot Token configured in credentials (not shown).  
    - Inputs: Reviews flagged by "Check Importance & Sentiment"  
    - Outputs: None  
    - Edge Cases: Telegram API errors, invalid Bot Token or Chat ID.

  - **Normal Notification (Telegram)**  
    - Type: Telegram node  
    - Role: Sends standard notification for other processed reviews with basic summary.  
    - Configuration:  
      - Text includes ReviewID, sentiment, importance, and category.  
      - Chat ID and Bot Token as above.  
    - Inputs: Reviews not flagged urgent  
    - Outputs: None  
    - Edge Cases: Same as Urgent Alert.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                           | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                                     |
|--------------------------|----------------------------------|-----------------------------------------|------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Overview    | Sticky Note                      | Documentation overview                   | None                   | None                          | ## 📋 AI Review Analyzer with Sentiment Analysis & Telegram Alerts...                                                         |
| Sticky Note - Step 1      | Sticky Note                      | Documentation for Step 1 block           | None                   | None                          | ### 1️⃣ Trigger & Data Fetch...                                                                                               |
| Sticky Note - Step 2      | Sticky Note                      | Documentation for Step 2 block           | None                   | None                          | ### 2️⃣ Filter & Format...                                                                                                    |
| Sticky Note - Step 3      | Sticky Note                      | Documentation for Step 3 block           | None                   | None                          | ### 3️⃣ AI Analysis...                                                                                                        |
| Sticky Note - Step 4      | Sticky Note                      | Documentation for Step 4 block           | None                   | None                          | ### 4️⃣ Save & Notify...                                                                                                      |
| Schedule Trigger         | Schedule Trigger                 | Starts workflow periodically             | None                   | Get Review Data               | ### 1️⃣ Trigger & Data Fetch...                                                                                               |
| Get Review Data          | Google Sheets                   | Fetches review data from Google Sheets   | Schedule Trigger       | Filter Unprocessed            | ### 1️⃣ Trigger & Data Fetch...                                                                                               |
| Filter Unprocessed       | If                             | Filters only unprocessed reviews          | Get Review Data        | Format Input Data             | ### 2️⃣ Filter & Format...                                                                                                    |
| Format Input Data        | Set                            | Formats data for AI input                 | Filter Unprocessed     | AI Agent - Review Analysis    | ### 2️⃣ Filter & Format...                                                                                                    |
| AI Agent - Review Analysis | langchain.agent                | Sends review to OpenAI GPT for analysis  | Format Input Data      | Parse AI Response             | ### 3️⃣ AI Analysis...                                                                                                        |
| OpenAI Chat Model        | langchain.lmChatOpenAi           | OpenAI GPT language model for agent      | AI Agent - Review Analysis | AI Agent - Review Analysis (internal) | ### 3️⃣ AI Analysis...                                                                                                        |
| Parse AI Response        | Code                           | Parses JSON from AI response, merges data| AI Agent - Review Analysis | Update Spreadsheet           | ### 3️⃣ AI Analysis...                                                                                                        |
| Update Spreadsheet       | Google Sheets                   | Updates analysis results back to sheet   | Parse AI Response      | Check Importance & Sentiment  | ### 4️⃣ Save & Notify...                                                                                                      |
| Check Importance & Sentiment | If                          | Routes based on importance or sentiment  | Update Spreadsheet     | Urgent Alert (Telegram), Normal Notification (Telegram) | ### 4️⃣ Save & Notify...                                                                                                      |
| Urgent Alert (Telegram)  | Telegram                       | Sends urgent alert for high priority/negative reviews | Check Importance & Sentiment | None                        | ### 4️⃣ Save & Notify...                                                                                                      |
| Normal Notification (Telegram) | Telegram                  | Sends standard notification for other reviews | Check Importance & Sentiment | None                        | ### 4️⃣ Save & Notify...                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Default interval (every minute or as desired)  
   - Connect output to next node.

2. **Create Google Sheets node ("Get Review Data")**  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID containing reviews  
   - Sheet Name: "Sheet1" (or your sheet’s gid)  
   - Connect input from Schedule Trigger.

3. **Create If node ("Filter Unprocessed")**  
   - Condition: Check if `ProcessStatus` field equals empty string ""  
   - Input: Connect from "Get Review Data"  
   - True output to next node.

4. **Create Set node ("Format Input Data")**  
   - Purpose: Pass through or optionally transform data to prepare for AI input  
   - Input: Connect from "Filter Unprocessed" True output.

5. **Create langchain.agent node ("AI Agent - Review Analysis")**  
   - Agent type: conversationalAgent  
   - System Message: "You are a multilingual review analysis expert. Accurately translate customer reviews and perform sentiment analysis and importance classification. Always respond ONLY in the specified JSON format."  
   - Prompt: Include detailed instructions for translation, sentiment (Positive/Neutral/Negative), sentiment score (-1.0 to +1.0), importance (High/Medium/Low) with definitions, category (Quality/Price/Shipping/Support/Features/Usability/Other), key phrase extraction, and specify exact JSON output format (see below).  
   - Connect input from "Format Input Data".

6. **Create langchain.lmChatOpenAi node ("OpenAI Chat Model")**  
   - Set temperature to 0.3  
   - Connect to AI Agent as the language model.

7. **Create Code node ("Parse AI Response")**  
   - Paste JavaScript code to extract JSON from AI response safely, parse it, and merge with original data, with fallback defaults on parse errors.  
   - Connect input from "AI Agent - Review Analysis".

8. **Create Google Sheets node ("Update Spreadsheet")**  
   - Operation: Update rows  
   - Document ID and Sheet Name same as in "Get Review Data"  
   - Mapping: Map fields ReviewID (used for matching), TranslatedText, Sentiment, SentimentScore, Importance, Category, KeyPhrase, ProcessStatus, and ProcessedAt from JSON data  
   - Cell format: USER_ENTERED  
   - Connect input from "Parse AI Response".

9. **Create If node ("Check Importance & Sentiment")**  
   - Condition: Importance equals "High" OR SentimentScore less than -0.5  
   - Connect input from "Update Spreadsheet".

10. **Create Telegram node ("Urgent Alert (Telegram)")**  
    - Action: Send message  
    - Message text: Detailed alert including ReviewID, original review, translated text, sentiment, sentiment score, importance, category, key phrase, and processed timestamp.  
    - Credentials: Telegram Bot Token and Chat ID configured.  
    - Connect input from "Check Importance & Sentiment" True output.

11. **Create Telegram node ("Normal Notification (Telegram)")**  
    - Action: Send message  
    - Message text: Summary notification with ReviewID, sentiment, importance, and category.  
    - Credentials: Same Telegram Bot Token and Chat ID.  
    - Connect input from "Check Importance & Sentiment" False output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow requires a Google Sheet structured with columns: ReviewID, Keyword (review text), ProcessStatus, plus optional columns for results.             | Setup Requirements section in Sticky Note - Overview                                            |
| OpenAI API credentials must be securely configured in n8n credentials for the langchain agent nodes.                                                          | n8n Credential management                                                                        |
| Telegram Bot Token and Chat ID must be set up and linked to the Telegram nodes for notifications to function properly.                                         | Telegram Bot creation and chat ID retrieval                                                     |
| Sentiment threshold (-0.5) for negative alerts can be adjusted in the "Check Importance & Sentiment" node to tune sensitivity.                               | Customization Tips in Sticky Note - Overview                                                    |
| AI prompt can be modified in "AI Agent - Review Analysis" node to adjust analysis criteria, output format, or language style.                                | Sticky Note - Overview                                                                           |
| This workflow uses LangChain nodes introduced in n8n v0.216+; ensure your n8n instance supports these nodes.                                                  | Version-specific requirement                                                                     |
| Parse AI Response node handles errors gracefully; however, unusual AI outputs may still require prompt tuning.                                                | Edge case mitigation                                                                             |
| Telegram alerts use markdown-style formatting for readability—ensure Telegram clients support this format.                                                   | Telegram nodes configuration                                                                    |
| For best performance, monitor API usage and rate limits on OpenAI and Google Sheets APIs.                                                                     | Operational best practices                                                                       |
| Workflow demo and similar examples can be found in n8n community forums and blog posts on AI-driven review analysis and alerting automation.                | https://community.n8n.io/ and https://n8n.io/blog/                                              |

---

**Disclaimer:** The text provided derives solely from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.