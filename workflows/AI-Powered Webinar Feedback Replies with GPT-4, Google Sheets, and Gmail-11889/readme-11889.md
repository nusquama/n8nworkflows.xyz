AI-Powered Webinar Feedback Replies with GPT-4, Google Sheets, and Gmail

https://n8nworkflows.xyz/workflows/ai-powered-webinar-feedback-replies-with-gpt-4--google-sheets--and-gmail-11889


# AI-Powered Webinar Feedback Replies with GPT-4, Google Sheets, and Gmail

### 1. Workflow Overview

This workflow automates the process of collecting webinar attendee feedback, analyzing it using AI, and sending personalized follow-up emails, while maintaining a log of interactions in Google Sheets. It is designed for webinar organizers who want to provide thoughtful, human-like responses at scale without manual intervention.

The workflow is logically divided into five main blocks:

- **1.1 Feedback Intake:** Receives attendee feedback via a webhook, generates a unique feedback ID, and normalizes the input data for consistent handling.
- **1.2 Data Persistence:** Stores the raw feedback along with generated IDs in Google Sheets and provides common webinar resources for AI contextualization.
- **1.3 AI Understanding:** Uses OpenAI GPT-4 to analyze the feedback, segment the attendee engagement level, and generate a structured response plan.
- **1.4 Personalized Follow-Up:** Builds a polished, professional HTML email based on the AI output and sends it via Gmail.
- **1.5 Feedback Log:** Records the final reply, email sending status, and timestamp back into Google Sheets for tracking and reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Feedback Intake

**Overview:**  
This block captures incoming feedback via a webhook, generates a human-readable unique ID for each feedback submission, and normalizes the data fields for downstream processing.

**Nodes Involved:**  
- Feedback Webhook  
- ID Generation  
- Normalize Feedback

**Node Details:**

- **Feedback Webhook**  
  - Type: Webhook  
  - Role: Entry point receiving HTTP POST requests containing attendee feedback from the webinar form.  
  - Configuration: Path set to `webinar-feedback`, HTTP method POST, response mode streaming.  
  - Connections: Output to ID Generation  
  - Edge Cases: Missing or malformed payload, webhook authorization issues (if enabled), slow responses can cause timeouts.

- **ID Generation**  
  - Type: Code (JavaScript)  
  - Role: Creates a unique, human-readable feedback ID combining attendee name and submission timestamp.  
  - Configuration: Extracts `Name` from the payload, sanitizes it by removing whitespace, and appends date and time in ISO format.  
  - Key Variables: `$json.body.Name`, `$json.submittedAt` or current date/time.  
  - Connections: Output to Normalize Feedback  
  - Edge Cases: Missing `Name` defaults to "Guest", date parsing errors.

- **Normalize Feedback**  
  - Type: Code (JavaScript)  
  - Role: Standardizes incoming feedback fields into consistent keys and formats.  
  - Configuration: Extracts Email, Name, Rating (as number), Feedback text (from multiple possible fields), Attended flag (default true), Webinar Title (default preset), and adds current timestamp `submittedAt`.  
  - Key Expressions: Handles missing fields with defaults, converts Rating to Number.  
  - Connections: Output to Store Partial  
  - Edge Cases: Missing or malformed fields, inconsistent feedback JSON structures.

---

#### 1.2 Data Persistence

**Overview:**  
Stores the normalized feedback data in Google Sheets for record-keeping and further processing. Enriches data with common webinar resources to assist AI in response generation.

**Nodes Involved:**  
- Store Partial  
- Common Resources  
- Merge (combines multiple data streams)

**Node Details:**

- **Store Partial**  
  - Type: Google Sheets  
  - Role: Appends new feedback entries into a Google Sheet with columns for participant info and original feedback.  
  - Configuration: Uses OAuth2 credentials, points to specific sheet and document IDs (placeholders in config). Maps normalized data fields with a generated `feedback_id`.  
  - Connections: Outputs to Merge node.  
  - Edge Cases: Google Sheets API rate limits, authentication failures, incorrect sheet or document IDs.

- **Common Resources**  
  - Type: Code (JavaScript)  
  - Role: Provides a static list of webinar-related resources (links to recording, slides, use-case library) as JSON for AI context.  
  - Configuration: Hardcoded array of label-url pairs.  
  - Connections: Output to Merge node.  
  - Edge Cases: None significant; static data.

- **Merge**  
  - Type: Merge  
  - Role: Combines three inputs: AI response, common resources, and partial stored feedback data into a single unified data object for downstream use.  
  - Configuration: Combine mode "combineByPosition" with 3 inputs.  
  - Connections: Output to Message a model (AI node).  
  - Edge Cases: Input misalignment if any branch does not produce outputs, which could cause merging errors.

---

#### 1.3 AI Understanding

**Overview:**  
This block leverages OpenAI GPT-4 to interpret attendee feedback, segment engagement level (hot/warm/cold), generate a natural, professional reply, and suggest subtle next steps and resources.

**Nodes Involved:**  
- Message a model  
- Parse AI Response

**Node Details:**

- **Message a model**  
  - Type: OpenAI GPT-4 (via LangChain)  
  - Role: Sends a prompt containing attendee feedback, rating, and webinar title to GPT-4 with detailed instructions for segmentation and reply generation.  
  - Configuration: Using model "gpt-4.1-mini". Prompt includes segmentation logic, writing guidelines, required JSON output format, and feedback context.  
  - Key Expressions: Uses expressions to insert feedback, rating, and webinar title dynamically from stored data.  
  - Connections: Output to Parse AI Response.  
  - Credentials: Requires valid OpenAI API key with GPT-4 access.  
  - Edge Cases: API rate limits, model unavailability, malformed prompt results, JSON parse errors in output.

- **Parse AI Response**  
  - Type: Code (JavaScript)  
  - Role: Extracts and safely parses the AI-generated JSON response to promote key fields (segment, thank you message, next steps) for later use.  
  - Configuration: Cleans raw AI text from markdown artifacts and attempts JSON parse; falls back to default "warm" segment with generic message on failure.  
  - Connections: Output to Merge node (for merging with other data).  
  - Edge Cases: Invalid JSON returned by AI, missing fields, empty responses.

---

#### 1.4 Personalized Follow-Up

**Overview:**  
Generates a clean, professional HTML email reply using the AI output and attendee data, then sends the email via Gmail.

**Nodes Involved:**  
- Build Email HTML  
- Send AI Thank You Email

**Node Details:**

- **Build Email HTML**  
  - Type: Code (JavaScript)  
  - Role: Constructs a styled HTML email including personalized greeting, AI-generated thank you message, optional next steps, and resource links.  
  - Configuration: Reads AI JSON from output, merges resources, and uses template literals for email structure and inline CSS styling.  
  - Connections: Output to Send AI Thank You Email.  
  - Edge Cases: Missing or malformed AI data, missing email or name fields, HTML encoding issues.

- **Send AI Thank You Email**  
  - Type: Gmail  
  - Role: Sends the constructed email to the attendee's email using Gmail OAuth2 credentials.  
  - Configuration: Sends to `$json.email`, subject and HTML content from previous node. Disables Gmail attribution footer.  
  - Credentials: Requires Gmail OAuth2 with send email permission.  
  - Connections: Output to Store Feedback (to log email sent status).  
  - Edge Cases: Gmail API rate limits, authentication expiration, network issues, invalid email addresses.

---

#### 1.5 Feedback Log

**Overview:**  
Logs the AI-generated reply, email sending status, and timestamp back into Google Sheets to maintain a complete record of interactions.

**Nodes Involved:**  
- Store Feedback

**Node Details:**

- **Store Feedback**  
  - Type: Google Sheets  
  - Role: Updates existing feedback rows with the AI reply message, email sent status, and sent timestamp for tracking.  
  - Configuration: Uses "update" operation keyed on `feedback_id`, writes columns "Our Reply", "Email Sent", and "Sent At". References values from merged data and Gmail output.  
  - Connections: None (end of flow).  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: API update failures, mismatched feedback_id, concurrent updates causing race conditions.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                        | Input Node(s)         | Output Node(s)             | Sticky Note                                                                                              |
|---------------------|-----------------------|-------------------------------------|-----------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Feedback Webhook     | Webhook               | Receive feedback HTTP POST entry    | -                     | ID Generation              | ## Step 1: Feedback Intake<br>Receives attendee responses, generates a readable feedback ID, and normalizes incoming form data. |
| ID Generation       | Code                  | Generate unique feedback ID          | Feedback Webhook      | Normalize Feedback         | ## Step 1: Feedback Intake<br>Receives attendee responses, generates a readable feedback ID, and normalizes incoming form data. |
| Normalize Feedback  | Code                  | Normalize and clean feedback data   | ID Generation          | Store Partial              | ## Step 1: Feedback Intake<br>Receives attendee responses, generates a readable feedback ID, and normalizes incoming form data. |
| Store Partial       | Google Sheets          | Append raw normalized feedback record | Normalize Feedback     | Merge                      | ## Step 2: Data Persistence<br>Stores raw feedback in Google Sheets and enriches it with shared webinar resources. |
| Common Resources    | Code                  | Provide static webinar resource links | -                     | Merge                      | ## Step 2: Data Persistence<br>Stores raw feedback in Google Sheets and enriches it with shared webinar resources. |
| Merge               | Merge                 | Combine AI, resources, and stored data | Store Partial, Message a model, Common Resources | Message a model            | ## Step 3: AI Understanding<br>Analyzes sentiment, intent, and engagement level, then produces a structured human-like response plan. |
| Message a model     | OpenAI GPT-4 (LangChain) | Analyze feedback and generate reply | Merge                  | Parse AI Response          | ## Step 3: AI Understanding<br>Analyzes sentiment, intent, and engagement level, then produces a structured human-like response plan. |
| Parse AI Response   | Code                  | Parse AI JSON response safely       | Message a model         | Merge                      | ## Step 3: AI Understanding<br>Analyzes sentiment, intent, and engagement level, then produces a structured human-like response plan. |
| Build Email HTML    | Code                  | Build personalized HTML email       | Merge                   | Send AI Thank You Email    | ## Step 4: Personalized Follow-Up<br>Builds a clean HTML email and sends a thoughtful, non-salesy response tailored to each attendee. |
| Send AI Thank You Email | Gmail               | Send email to attendee               | Build Email HTML        | Store Feedback             | ## Step 4: Personalized Follow-Up<br>Builds a clean HTML email and sends a thoughtful, non-salesy response tailored to each attendee. |
| Store Feedback     | Google Sheets          | Update feedback record with email status | Send AI Thank You Email | -                          | ## Step 5: Feedback Log<br>Writes the sent email, timestamp, and status back to Google Sheets for visibility and reporting. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set HTTP Method: POST  
   - Path: `webinar-feedback`  
   - Response Mode: Streaming  
   - Purpose: Entry point for receiving feedback submissions.

2. **Add Code Node: ID Generation**  
   - Mode: Run Once for Each Item  
   - JavaScript: Generate `feedback_id` by concatenating sanitized attendee name and submission ISO date/time.  
   - Connect output of Webhook node to this node.

3. **Add Code Node: Normalize Feedback**  
   - Mode: Run Once for Each Item  
   - JavaScript: Extract and normalize fields: email, name (default "Attendee"), rating (number), feedback text, attended flag (default true), webinar title (default "AI Automation Mastery Webinar"), and submission timestamp (ISO now).  
   - Connect from ID Generation output.

4. **Add Google Sheets Node: Store Partial**  
   - Operation: Append  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: Your Sheet Name  
   - Columns mapped: Name, Email, Rating, feedback_id, Webinar Title, Original Feedback  
   - Credentials: Google Sheets OAuth2  
   - Connect from Normalize Feedback output.

5. **Add Code Node: Common Resources**  
   - JavaScript: Return an array of JSON objects with labels and URLs for webinar recording, slides, and use-case library.  
   - No input connections.

6. **Add Merge Node**  
   - Mode: Combine  
   - Combine by Position  
   - Number of Inputs: 3  
   - Connect inputs:  
     - Input 1: Output from Store Partial  
     - Input 2: Output from Message a model (to be created)  
     - Input 3: Output from Common Resources

7. **Add OpenAI Node: Message a model**  
   - Type: OpenAI GPT-4 via LangChain node  
   - Model: `gpt-4.1-mini`  
   - Prompt: Insert detailed prompt with instructions for segmentation, response style, output JSON format, and feedback data placeholders from Store Partial.  
   - Credentials: OpenAI API Key with GPT-4 access  
   - Connect from Merge node (partial feedback and resources injection).  
   - Output connects to Parse AI Response.

8. **Add Code Node: Parse AI Response**  
   - Mode: Run Once for Each Item  
   - JavaScript: Extract raw AI text, remove markdown code blocks, parse JSON safely, fallback to default segment and message if parsing fails.  
   - Connect from Message a model output.  
   - Output connects back to Merge node to combine AI data.

9. **Add Code Node: Build Email HTML**  
   - Mode: Run Once for Each Item  
   - JavaScript: Use attendee name, webinar title, AI thank you message, next steps, and resources to build a styled HTML email string with inline CSS.  
   - Connect from merged AI response and data output.

10. **Add Gmail Node: Send AI Thank You Email**  
    - Send To: `{{$json.email}}`  
    - Subject: `Thank you for your feedback – {{$json['Webinar Title']}}`  
    - Message: HTML content from previous node `{{$json.htmlContent}}`  
    - Options: Disable Gmail attribution  
    - Credentials: Gmail OAuth2 with send email scope  
    - Connect from Build Email HTML node.

11. **Add Google Sheets Node: Store Feedback**  
    - Operation: Update  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: Your Sheet Name  
    - Matching Columns: feedback_id  
    - Columns updated: Our Reply (AI thank you message), Email Sent (email sent status), Sent At (timestamp)  
    - Credentials: Google Sheets OAuth2  
    - Connect from Gmail send node output.

12. **Activate the Workflow**  
    - Test with sample feedback POST requests.  
    - Confirm data flows correctly, emails send, and logs update appropriately.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| This workflow is ideal for webinars, workshops, and live events where personalized, high-quality follow-up matters.                                                                                                            | Branding and general description (Sticky Note covering workflow) |
| Setup requires valid OAuth2 credentials for Google Sheets and Gmail, plus OpenAI API key with GPT-4 access.                                                                                                                   | Credential requirements                                           |
| Webinar resources URLs (recording, slides, use-cases) must be updated in the "Common Resources" code node to reflect your actual content.                                                                                      | Common Resources node configuration                              |
| The AI segmentation logic internally classifies feedback as HOT, WARM, or COLD based on rating and attendance, but this is not disclosed in the email reply.                                                                  | AI prompt detailed in Message a model node                      |
| Emails are built with responsive, accessible HTML and inline CSS to ensure compatibility across major email clients.                                                                                                         | Build Email HTML node code                                        |
| Logs are maintained in Google Sheets for audit, analytics, and manual review if necessary.                                                                                                                                     | Data persistence nodes                                           |
| Links in email resources open in new tabs (`target="_blank"`) for better user experience.                                                                                                                                      | Email HTML template                                               |
| The workflow handles JSON parsing errors gracefully by falling back to default messages to avoid crashes or incomplete emails.                                                                                                | Parse AI Response node                                            |
| For advanced usage, the workflow can be extended to branch responses or trigger additional workflows based on AI segmentation.                                                                                                | Extendability note                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a low-code integration and automation platform. This processing strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.