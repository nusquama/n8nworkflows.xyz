AI Lead Scoring & Routing with Google Gemini, Sheets and Slack Notifications

https://n8nworkflows.xyz/workflows/ai-lead-scoring---routing-with-google-gemini--sheets-and-slack-notifications-8985


# AI Lead Scoring & Routing with Google Gemini, Sheets and Slack Notifications

### 1. Workflow Overview

This workflow automates lead intake, sentiment classification, storage, and notification for lead management using AI and integration tools. It targets businesses or teams who receive leads via Typeform and want to automatically score and route these leads based on sentiment analysis. The workflow groups leads into Hot, Neutral, or Cold segments, stores them in Google Sheets accordingly, and sends notifications to a Slack channel.

Logical blocks include:  
- **1.1 Input Reception:** Receiving new leads from Typeform via a webhook and preparing lead data.  
- **1.2 AI Sentiment Classification:** Using Google Gemini AI model (via Langchain node) to classify lead sentiment (Hot, Neutral, Cold).  
- **1.3 Data Storage:** Storing leads into Google Sheets tabs based on sentiment classification.  
- **1.4 Notification:** Combining stored lead data and sending a Slack notification summarizing the lead.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives incoming leads via a Typeform webhook, then cleans and maps the lead data for downstream processing.  
- **Nodes Involved:**  
  - Receive New Lead (Typeform)  
  - Prepare Lead Data  
  - Sticky Note (STEP 1 · Intake)  

- **Node Details:**  

  - **Receive New Lead (Typeform)**  
    - Type: Webhook  
    - Role: Entry point for new leads submitted through Typeform.  
    - Configuration: HTTP POST webhook at path `/lead-webhook`. Expects JSON payload from Typeform.  
    - Inputs: External HTTP POST request.  
    - Outputs: Raw lead data JSON.  
    - Edge cases: Missing or malformed payload, incorrect HTTP method, webhook URL misconfiguration.  

  - **Prepare Lead Data**  
    - Type: Set (data transformation)  
    - Role: Cleans or maps incoming lead fields (e.g., name, email, message, timestamp) to standardized keys for later use.  
    - Configuration: Uses expressions to extract necessary fields from the webhook JSON. No explicit fields shown in JSON but implied by sticky note.  
    - Inputs: Raw webhook data from previous node.  
    - Outputs: Formatted lead data JSON.  
    - Edge cases: Missing expected fields, inconsistent field names causing expression failures.  

  - **Sticky Note (STEP 1 · Intake)**  
    - Role: Documentation within workflow for user clarity.  
    - Content: Explains the webhook setup and data preparation.

---

#### 1.2 AI Sentiment Classification

- **Overview:** Analyzes the lead message text to classify the sentiment using Google Gemini AI via Langchain integration, outputting a sentiment category that drives routing.  
- **Nodes Involved:**  
  - Classify Sentiment (Gemini or other AI model)  
  - Google Gemini Chat Model  
  - Sticky Note1 (STEP 2 · Sentiment Classification)  

- **Node Details:**  

  - **Classify Sentiment (Gemini or other AI model)**  
    - Type: Langchain Sentiment Analysis node  
    - Role: Runs sentiment classification on the lead message text.  
    - Configuration: Input text expression extracts message from lead data using fallbacks: `$json["message"] || $json["mensagem"] || $json["resposta"]`.  
    - Inputs: Prepared lead data from previous block, and AI model output from Google Gemini Chat Model node.  
    - Outputs: Branches on sentiment: Hot, Neutral, Cold (3 outputs).  
    - Edge cases: Missing or empty message text, AI API errors (timeouts, auth), unexpected output format.  

  - **Google Gemini Chat Model**  
    - Type: Langchain Google Gemini LLM node  
    - Role: Provides AI model integration for classification requests.  
    - Configuration: Uses Google PaLM API credentials, no additional options specified.  
    - Inputs: Triggered internally by Langchain sentiment node.  
    - Outputs: AI-generated sentiment classification.  
    - Edge cases: API quota exceeded, invalid credentials, network failure.  

  - **Sticky Note1 (STEP 2 · Sentiment Classification)**  
    - Role: Explains how sentiment classification is performed and how outputs are used for routing.

---

#### 1.3 Data Storage

- **Overview:** Stores leads into separate Google Sheets tabs or rows based on their sentiment classification to facilitate segmented lead management.  
- **Nodes Involved:**  
  - Store Hot Lead  
  - Store Neutral Lead  
  - Store Cold Lead  
  - Sticky Note2 (STEP 3 · Store by Segment)  

- **Node Details:**  

  For each of the three storage nodes (Hot, Neutral, Cold):  
  - Type: Google Sheets node  
  - Role: Append or update lead data into a Google Sheet tab depending on sentiment branch.  
  - Configuration:  
    - Operation: Append or update  
    - Sheet Name: Default tab (gid=0) in the target spreadsheet  
    - Document ID: Specific Google Sheet for Founders Academy Waitlist  
    - Columns: Defined schema includes Full Name, First Name, Last Name, Email, Phone, Submitted date  
    - Matching Columns: Empty (no matching columns set to avoid duplicates, which could lead to duplicates)  
  - Inputs: Lead data routed by sentiment classification node.  
  - Outputs: Lead data forwarded for merging downstream.  
  - Edge cases: Google Sheets API errors, permission issues, rate limits, missing or malformed data fields, duplicate records due to missing matching columns.  

  - **Sticky Note2 (STEP 3 · Store by Segment)**  
    - Role: Instructs on setting matchingColumns to avoid duplicates and mapping columns explicitly.

---

#### 1.4 Notification

- **Overview:** Merges all sentiment branches back to a single stream and sends a formatted notification message to a Slack channel alerting the team about the new lead and its classification.  
- **Nodes Involved:**  
  - Combine Lead Data (Merge node)  
  - Send a message (Slack node)  
  - Sticky Note3 (STEP 4 · Combine & Notify)  

- **Node Details:**  

  - **Combine Lead Data**  
    - Type: Merge (multiple inputs)  
    - Role: Combines three inputs (Hot, Neutral, Cold leads) into one output.  
    - Configuration: Number of inputs set to 3, merging by appending.  
    - Inputs: Data streams from Store Hot Lead, Store Neutral Lead, Store Cold Lead nodes.  
    - Outputs: Combined lead data for notification.  
    - Edge cases: Missing inputs if one sentiment branch has no leads, merge failures if data formats mismatch.  

  - **Send a message (Slack)**  
    - Type: Slack node  
    - Role: Posts a notification message to a Slack channel about the new lead and sentiment.  
    - Configuration:  
      - Authentication: OAuth2 with Slack account credentials  
      - Channel: Hardcoded channel ID (C09BP1AGRT9) selected from cached list  
      - Text: Placeholder text "wefwe" (requires replacement with formatted summary including lead name, sentiment, link)  
    - Inputs: Combined lead data from Merge node.  
    - Outputs: None (terminal node).  
    - Edge cases: Slack authentication failure, invalid channel ID, API rate limits, missing or unformatted message text.  

  - **Sticky Note3 (STEP 4 · Combine & Notify)**  
    - Role: Explains the purpose of merging and Slack notification, including a tip to replace placeholder text with a formatted message.

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                    | Input Node(s)                                   | Output Node(s)                  | Sticky Note                                                                                                         |
|-------------------------------|-------------------------------|----------------------------------|------------------------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Receive New Lead (Typeform)    | Webhook                       | Lead intake entry point          | External HTTP POST                             | Prepare Lead Data              | ## STEP 1 · Intake (Webhook)\n**Receive New Lead (Typeform):** POST /lead-webhook\n**Prepare Lead Data:** clean/map fields you need (name, email, message, timestamp).\nTip: Ensure the Typeform payload keys match what you reference later. |
| Prepare Lead Data              | Set                           | Clean and map lead fields        | Receive New Lead (Typeform)                     | Classify Sentiment (Gemini or other ai model) | ## STEP 1 · Intake (Webhook)\n**Receive New Lead (Typeform):** POST /lead-webhook\n**Prepare Lead Data:** clean/map fields you need (name, email, message, timestamp).\nTip: Ensure the Typeform payload keys match what you reference later. |
| Google Gemini Chat Model       | Langchain Google Gemini LLM   | AI model integration             | (Triggered internally by Langchain node)       | Classify Sentiment (Gemini or other ai model) | ## STEP 2 · Sentiment Classification\n**Sentiment node** analyzes message text (fallbacks: message | mensagem | resposta).\n**Gemini model** (via LLM node) powers the classifier.\nOutput branches: Hot / Neutral / Cold (used for routing). |
| Classify Sentiment (Gemini or other ai model) | Langchain Sentiment Analysis | Sentiment classification and routing | Prepare Lead Data, Google Gemini Chat Model     | Store Hot Lead, Store Neutral Lead, Store Cold Lead | ## STEP 2 · Sentiment Classification\n**Sentiment node** analyzes message text (fallbacks: message | mensagem | resposta).\n**Gemini model** (via LLM node) powers the classifier.\nOutput branches: Hot / Neutral / Cold (used for routing). |
| Store Hot Lead                | Google Sheets                 | Store Hot leads in spreadsheet  | Classify Sentiment (Gemini or other ai model)  | Combine Lead Data             | ## STEP 3 · Store by Segment (Sheets)\nEach branch writes to Google Sheets (appendOrUpdate).\nTip: Set **matchingColumns** (e.g., Email or Submitted) to avoid duplicates.\nIf no mapping is defined, enable auto-map or define column mapping explicitly. |
| Store Neutral Lead            | Google Sheets                 | Store Neutral leads in spreadsheet | Classify Sentiment (Gemini or other ai model)  | Combine Lead Data             | ## STEP 3 · Store by Segment (Sheets)\nEach branch writes to Google Sheets (appendOrUpdate).\nTip: Set **matchingColumns** (e.g., Email or Submitted) to avoid duplicates.\nIf no mapping is defined, enable auto-map or define column mapping explicitly. |
| Store Cold Lead               | Google Sheets                 | Store Cold leads in spreadsheet | Classify Sentiment (Gemini or other ai model)  | Combine Lead Data             | ## STEP 3 · Store by Segment (Sheets)\nEach branch writes to Google Sheets (appendOrUpdate).\nTip: Set **matchingColumns** (e.g., Email or Submitted) to avoid duplicates.\nIf no mapping is defined, enable auto-map or define column mapping explicitly. |
| Combine Lead Data             | Merge                        | Combine all sentiment branches   | Store Hot Lead, Store Neutral Lead, Store Cold Lead | Send a message              | ## STEP 4 · Combine & Notify\n**Merge (3 inputs)** recombines Hot/Neutral/Cold.\n**Slack message** posts to the channel.\nTip: Replace placeholder text with a formatted summary (name, sentiment, link to row). |
| Send a message                | Slack                        | Send Slack notification          | Combine Lead Data                               | None                          | ## STEP 4 · Combine & Notify\n**Merge (3 inputs)** recombines Hot/Neutral/Cold.\n**Slack message** posts to the channel.\nTip: Replace placeholder text with a formatted summary (name, sentiment, link to row). |
| Sticky Note                  | Sticky Note                  | Documentation                   | None                                           | None                          | ## STEP 1 · Intake (Webhook)\n**Receive New Lead (Typeform):** POST /lead-webhook\n**Prepare Lead Data:** clean/map fields you need (name, email, message, timestamp).\nTip: Ensure the Typeform payload keys match what you reference later. |
| Sticky Note1                 | Sticky Note                  | Documentation                   | None                                           | None                          | ## STEP 2 · Sentiment Classification\n**Sentiment node** analyzes message text (fallbacks: message | mensagem | resposta).\n**Gemini model** (via LLM node) powers the classifier.\nOutput branches: Hot / Neutral / Cold (used for routing). |
| Sticky Note2                 | Sticky Note                  | Documentation                   | None                                           | None                          | ## STEP 3 · Store by Segment (Sheets)\nEach branch writes to Google Sheets (appendOrUpdate).\nTip: Set **matchingColumns** (e.g., Email or Submitted) to avoid duplicates.\nIf no mapping is defined, enable auto-map or define column mapping explicitly. |
| Sticky Note3                 | Sticky Note                  | Documentation                   | None                                           | None                          | ## STEP 4 · Combine & Notify\n**Merge (3 inputs)** recombines Hot/Neutral/Cold.\n**Slack message** posts to the channel.\nTip: Replace placeholder text with a formatted summary (name, sentiment, link to row). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Receive New Lead (Typeform)**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `lead-webhook`  
   - Purpose: Entry point for incoming lead data from Typeform.  

2. **Create Set Node: Prepare Lead Data**  
   - Connect from webhook node.  
   - Purpose: Map and clean incoming data fields to consistent keys (e.g., name, email, message, timestamp).  
   - Configuration: Use expressions to extract fields; ensure compatibility with Typeform payload structure.  

3. **Create Langchain Google Gemini Chat Model Node**  
   - Credentials: Configure with Google PaLM API credentials (Google Gemini).  
   - No custom options needed.  

4. **Create Langchain Sentiment Analysis Node: Classify Sentiment**  
   - Connect input from “Prepare Lead Data” and set AI model to the Google Gemini node created above.  
   - Input Text Expression: `{{$json["message"] || $json["mensagem"] || $json["resposta"]}}`  
   - Configure node to branch outputs into three: Hot, Neutral, Cold.  

5. **Create Three Google Sheets Nodes:**  
   - Names: Store Hot Lead, Store Neutral Lead, Store Cold Lead (one per sentiment).  
   - Operation: Append or update rows.  
   - Document ID: Link to your Google Sheet (e.g., Founders Academy Waitlist).  
   - Sheet Name: Default or specify tab (gid=0).  
   - Columns: Map lead fields (Full Name, First Name, Last Name, Email, Phone, Submitted).  
   - Matching Columns: (Optional but recommended) Set at least Email or Submitted date to prevent duplicates.  
   - Connect each node to corresponding output from Sentiment Classification node (Hot -> Store Hot Lead, etc.).  

6. **Create Merge Node: Combine Lead Data**  
   - Number of Inputs: 3 (one for each sentiment storage node).  
   - Connect outputs of Store Hot Lead, Store Neutral Lead, Store Cold Lead nodes to this merge node.  

7. **Create Slack Node: Send a message**  
   - Connect from Merge node.  
   - Authentication: Configure with Slack OAuth2 credentials.  
   - Channel: Select or enter channel ID (e.g., `C09BP1AGRT9`).  
   - Message: Replace placeholder text with formatted lead summary including name, sentiment, and a link to the Google Sheet row if possible.  

8. **Add Sticky Notes for Documentation (optional but recommended):**  
   - Add sticky notes near each logical block describing their purpose (as per steps 1 to 4).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Tip: Matching columns in Google Sheets nodes should be set (e.g., Email or Submitted) to avoid duplicate records in the spreadsheet.       | See Sticky Note2 in workflow for detailed advice on storage node configuration.                    |
| Slack notification text is currently a placeholder; replace it with a formatted message summarizing lead data and sentiment for clarity.   | See Sticky Note3 for notification best practices.                                                 |
| The Google Gemini Chat Model node requires valid Google PaLM API credentials; ensure quota and authentication are properly set up.         | Workflow credential configuration; Google Gemini API documentation: https://developers.generativeai.google/ |
| Typeform webhook payload keys must match expressions in the Prepare Lead Data node to avoid data mapping issues.                           | Sticky Note STEP 1 · Intake advises to verify Typeform payload structure.                          |

---

**Disclaimer:** The content is generated based solely on the provided n8n workflow JSON and adheres to content policies. It contains no illegal or protected information. All data handled is legal and public.