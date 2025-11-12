Analyze BeyondPresence Video Calls with GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/analyze-beyondpresence-video-calls-with-gpt-4o-mini-and-google-sheets-4436


# Analyze BeyondPresence Video Calls with GPT-4o-mini and Google Sheets

### 1. Workflow Overview

This workflow automates the analysis of video calls recorded by BeyondPresence, leveraging GPT-4o-mini AI for conversational insights and storing results in Google Sheets for easy access and reporting.

**Target Use Cases:**  
- Post-call analysis for video agent interactions  
- Automated extraction of call summaries, sentiments, and action items  
- Centralized storage of call analytics data for team review and follow-up tracking  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives call data via a webhook triggered when a BeyondPresence video call ends.  
- **1.2 Initial Validation & Enrichment:** Validates webhook data integrity and enriches with calculated metadata (e.g., call duration, time of day).  
- **1.3 AI Processing:** Sends enriched call data to GPT-4o-mini for detailed analysis and extracts structured JSON response.  
- **1.4 Data Storage:** Appends analyzed call results into a configured Google Sheets document for record keeping.  
- **1.5 Optional Error Handling & Extensions:** Suggested nodes for error management and further integrations such as Slack notifications or CRM updates (documented but inactive).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures the BeyondPresence call end event via webhook, acknowledges receipt to the source system, and filters events to process only relevant call-ended types.

**Nodes Involved:**  
- BeyondPresence Webhook  
- Acknowledge Webhook  
- Filter Call Events

**Node Details:**  

- **BeyondPresence Webhook**  
  - Type: Webhook Trigger  
  - Configuration: Listens on path `/beyondpresence-call-webhook` for HTTP POST events. Raw body parsing disabled.  
  - Inputs: External BeyondPresence system webhook calls  
  - Outputs: JSON payload containing call metadata, transcript, and evaluation metrics  
  - Edge cases: Missing or malformed HTTP payloads; network interruptions; webhook security not detailed here.  

- **Acknowledge Webhook**  
  - Type: Respond to Webhook  
  - Configuration: Returns HTTP 200 with JSON confirmation including status, message, and timestamp based on current time.  
  - Inputs: Output from webhook node  
  - Outputs: HTTP response sent back to webhook caller  
  - Edge cases: Delay in response could cause webhook retries; ensure response sent before downstream processing.  

- **Filter Call Events**  
  - Type: Switch  
  - Configuration: Checks if `body.event_type` equals `"call_ended"`. Routes matching events to processing; others to fallback output.  
  - Inputs: JSON from acknowledge webhook  
  - Outputs: Valid call ended events forwarded; others ignored or handled separately  
  - Edge cases: Event type missing or unexpected; configured to continue on fail to avoid blocking workflow.

---

#### 2.2 Initial Validation & Enrichment

**Overview:**  
Validates the presence and structure of critical call data fields, calculates call duration and contextual metadata, and identifies presence of action items in messages.

**Nodes Involved:**  
- Validate & Enrich Data

**Node Details:**  

- **Validate & Enrich Data**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Checks for required fields: `event_type`, `call_id`, `call_data` (with `userName`, `startedAt`, `endedAt`), and `messages` array.  
    - Throws error if validation fails, halting processing for that input.  
    - Calculates:  
      - Call duration in minutes and formatted string  
      - Day of week from call start time  
      - Time of day classification (Early Morning, Morning, Afternoon, Evening, Night)  
      - Message count  
      - Boolean flag if messages include action items (keywords like "will do", "follow up")  
  - Inputs: JSON call data from Filter Call Events node  
  - Outputs: JSON enriched with validation status and calculated fields  
  - Edge cases: Missing or malformed timestamps; empty or non-array messages; logic errors in keyword detection; fails gracefully with error messages.

---

#### 2.3 AI Processing

**Overview:**  
Uses OpenAI GPT-4o-mini to analyze the enriched call data and generate a structured JSON summary including sentiment, main points, action items, and recommendations.

**Nodes Involved:**  
- AI Call Analysis  
- Parse AI Response

**Node Details:**  

- **AI Call Analysis**  
  - Type: LangChain OpenAI Node  
  - Configuration:  
    - Model: `gpt-4o-mini`  
    - Temperature: 0.3, Top-p: 0.9 for balanced creativity and determinism  
    - System message: Detailed instructions on analysis requirements and strict JSON output format  
    - User message: Injects call data, evaluation, and messages as stringified JSON for context  
  - Inputs: Validated and enriched JSON from previous node  
  - Outputs: AI-generated message content with JSON embedded in markdown code block  
  - Edge cases: API authentication errors; rate limiting; malformed AI output; timeout or incomplete response; dependency on accurate prompt formatting.

- **Parse AI Response**  
  - Type: Code (JavaScript)  
  - Configuration:  
    - Extracts JSON content from AI response within markdown code blocks or directly from content if code block missing  
    - Cleans JSON string from comments, trailing commas, and fixes common JSON formatting issues before parsing  
    - Throws informative errors if parsing fails  
  - Inputs: Raw AI text response  
  - Outputs: Parsed JSON object conforming to analysis schema  
  - Edge cases: AI returns incomplete or invalid JSON; unexpected formatting; fallback parsing failures.

---

#### 2.4 Data Storage

**Overview:**  
Appends the structured call analysis data into a preconfigured Google Sheets document for archival and reporting.

**Nodes Involved:**  
- Save to Google Sheets

**Node Details:**  

- **Save to Google Sheets**  
  - Type: Google Sheets Node  
  - Configuration:  
    - Operation: Append row  
    - Document ID: User must specify by copying provided template sheet and entering new sheet ID  
    - Sheet Name: Uses first sheet (`gid=0`)  
    - Columns mapped to analysis fields: Date, Title, Topic, Summary, Follow-ups, Participant, References, Action Items, Main Points, Sentiment info, and more (total ~30 columns)  
    - Cell format: USER_ENTERED to allow formulas and formatting  
  - Inputs: Parsed JSON from AI analysis  
  - Outputs: Confirmation of row append  
  - Edge cases: Invalid or missing sheet ID; insufficient permissions; API quotas; schema mismatch if template altered; network issues.

---

#### 2.5 Optional Error Handling & Extensions (Documented Notes)

**Overview:**  
Provides guidance on optional enhancements like error logging, Slack notifications, CRM integrations, and database backups to extend workflow functionality.

**Nodes Involved:**  
- Error Handling Note (sticky note)  
- Extension Options (sticky note)

**Node Details:**  

- **Error Handling Note**  
  - Type: Sticky Note  
  - Content: Suggests adding an error trigger node to capture failures, log to separate sheet, notify admins, and retry failed operations.  
  - Not connected in current workflow, serves as best-practice advice.  

- **Extension Options**  
  - Type: Sticky Note  
  - Content: Recommends integrations for Slack notifications on sentiment/action items, database backups for analytics, and CRM updates for call records.  
  - Not implemented but provides roadmap for future enhancements.

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                   | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                       |
|---------------------------|------------------------------|---------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Workflow Overview         | Sticky Note                  | Overview and instructions        | —                            | —                           | Describes webhook trigger purpose and setup instructions                                        |
| Processing Steps          | Sticky Note                  | Explains validation and AI steps| —                            | —                           | Details data validation, AI analysis, and parsing logic                                         |
| Data Storage              | Sticky Note                  | Google Sheets setup instructions| —                            | —                           | Provides quick start guide and link to template sheet                                           |
| BeyondPresence Webhook    | Webhook                      | Receives call end event          | External webhook              | Acknowledge Webhook          | See Workflow Overview note for webhook config instructions                                      |
| Acknowledge Webhook       | Respond to Webhook           | Sends 200 OK response            | BeyondPresence Webhook        | Filter Call Events           | Same as above                                                                                   |
| Filter Call Events        | Switch                      | Routes only call_ended events    | Acknowledge Webhook           | Validate & Enrich Data       | Validates event type filtering                                                                 |
| Validate & Enrich Data    | Code (JavaScript)            | Validates and enriches data      | Filter Call Events            | AI Call Analysis            | Handles data integrity and adds calculated fields                                               |
| AI Call Analysis          | LangChain OpenAI             | Generates AI summary             | Validate & Enrich Data        | Parse AI Response            | Uses GPT-4o-mini with detailed prompt                                                          |
| Parse AI Response         | Code (JavaScript)            | Parses AI JSON output            | AI Call Analysis              | Save to Google Sheets        | Extracts and cleans JSON from AI response                                                      |
| Save to Google Sheets     | Google Sheets                | Stores analysis results          | Parse AI Response             | —                           | Requires Google Sheets ID; maps AI data to columns                                             |
| Error Handling Note       | Sticky Note                  | Suggests error handling          | —                            | —                           | Advises adding error triggers and logging                                                      |
| Extension Options         | Sticky Note                  | Suggests additional integrations| —                            | —                           | Lists optional Slack, database, CRM integrations                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create BeyondPresence Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `beyondpresence-call-webhook`  
   - Raw Body: Disabled  
   - Response Mode: Response Node (to allow separate response node)  

2. **Create Acknowledge Webhook Node**  
   - Type: Respond to Webhook  
   - Connect input to BeyondPresence Webhook output  
   - Response Code: 200  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "status": "success",
       "message": "Webhook received and processing",
       "timestamp": "{{ $now.toISO() }}"
     }
     ```

3. **Create Filter Call Events Node**  
   - Type: Switch  
   - Connect input to Acknowledge Webhook output  
   - Add rule:  
     - Condition: `$json.body.event_type == "call_ended"` (case sensitive)  
     - Output key: `call_ended`  
   - Fallback output: `extra`  
   - Continue on fail: Enabled

4. **Create Validate & Enrich Data Node**  
   - Type: Code (JavaScript)  
   - Connect input to Filter Call Events `call_ended` output  
   - Paste provided JS code that:  
     - Validates required fields in `body`  
     - Calculates call duration, day/time info, message count, action items presence  
     - Throws error if validation fails  
   - Continue on fail: Enabled (optional but recommended)

5. **Create AI Call Analysis Node**  
   - Type: LangChain OpenAI  
   - Connect input to Validate & Enrich Data output  
   - Model ID: `gpt-4o-mini`  
   - Temperature: 0.3  
   - Top-p: 0.9  
   - Messages:  
     - System message with detailed instructions and JSON output format (use provided prompt with dynamic expressions)  
     - User message embedding call data, evaluation, and messages as JSON strings  
   - Credential: Configure OpenAI or compatible API credential

6. **Create Parse AI Response Node**  
   - Type: Code (JavaScript)  
   - Connect input to AI Call Analysis output  
   - Paste provided JS code that extracts JSON from the AI response content, cleans and parses it  
   - Continue on fail: Enabled (optional but recommended)

7. **Create Save to Google Sheets Node**  
   - Type: Google Sheets  
   - Connect input to Parse AI Response output  
   - Operation: Append  
   - Document ID: Enter your copied Google Sheet ID  
   - Sheet Name: Use first sheet (`gid=0`)  
   - Map columns according to JSON fields: Date, Title, Topic, Summary, Follow-ups, Participant, Action Items, Sentiment, etc. (map all fields as per node configuration)  
   - Credential: Configure Google Sheets OAuth2 credentials  
   - Cell Format: USER_ENTERED

8. **Add Sticky Notes for Documentation** (optional but recommended)  
   - Create notes for Workflow Overview, Processing Steps, Data Storage, Error Handling, and Extension Options with the provided content for clarity.

9. **Test and Activate Workflow**  
   - Deploy the workflow and test with a sample BeyondPresence webhook POST containing a call_ended event with full call data.  
   - Confirm successful AI analysis and row append in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                |
|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Copy the Google Sheets template to your drive before using and update the node with your sheet ID.                 | https://docs.google.com/spreadsheets/d/1TO6-jkCtoSFNLJObtN0UyklgdUd3ZxEnUaNvUaBjpvo/copy |
| Webhook URL must be configured in BeyondPresence dashboard under Settings → Webhooks to ensure event delivery.      | BeyondPresence platform configuration                          |
| AI prompt enforces strict JSON output for reliable parsing and downstream processing.                               | Embedded in AI Call Analysis node system message               |
| Recommended extensions include Slack notifications, database backups, and CRM integration for enhanced workflows. | See Extension Options sticky note                              |
| Suggested error handling includes logging errors to a sheet and notifying admins for smooth operational robustness.| See Error Handling Note sticky note                            |

---

This completes the comprehensive reference document for the "BeyondPresence Video Agent Call Analytics with AI & Google Sheets" workflow, enabling advanced users and AI agents to understand, reproduce, and extend the automation reliably.