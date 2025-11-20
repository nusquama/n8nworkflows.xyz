Automate Content Publishing with GPT-4 via Google Sheets to Email & Slack Approval

https://n8nworkflows.xyz/workflows/automate-content-publishing-with-gpt-4-via-google-sheets-to-email---slack-approval-10830


# Automate Content Publishing with GPT-4 via Google Sheets to Email & Slack Approval

### 1. Workflow Overview

This workflow automates the publishing pipeline for content created and optimized in Google Sheets by leveraging GPT-4 AI capabilities. It is designed to:

- Fetch draft content from Google Sheets.
- Use an AI agent to enhance, format, and generate SEO metadata.
- Store the enriched content back into Sheets.
- Route the content through an email-based approval process.
- Upon approval, publish the content via email to a recipient and notify via Slack.

The workflow is structured into the following logical blocks:

- **1.1 Trigger Section:** Entry point via chat message to start the publishing process with initial metadata preparation.
- **1.2 AI Processing:** Fetch the draft, enrich it with AI (formatting, SEO metadata), and produce structured JSON output.
- **1.3 Storage:** Save the AI-processed content and metadata back into Google Sheets for tracking.
- **1.4 Approval Process:** Send content for human approval via Gmail, check approval status, and upon approval, publish content and notify Slack.
- **1.5 Security & Setup Notes:** Credential requirements and setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Section

- **Overview:**  
  This block initiates the workflow upon receiving a chat message trigger. It prepares essential publishing metadata such as content ID, topic, intent, and platform parameters for downstream processing.

- **Nodes Involved:**  
  - When chat message received  
  - Prepare Publishing Metadata

- **Node Details:**

  - **When chat message received**  
    - Type: Langchain Chat Trigger  
    - Role: Entry point listening for chat messages to trigger the workflow.  
    - Configuration: Default settings; webhook ID assigned.  
    - Inputs: External chat message webhook.  
    - Outputs: Emits a trigger with initial data for metadata preparation.  
    - Failure modes: Network/webhook failures, malformed input.  
    - Version: 1.3.

  - **Prepare Publishing Metadata**  
    - Type: Set node  
    - Role: Sets key metadata variables for publishing: `intent` (fixed "publish"), `topic` (fixed "AI Seo Basics"), `content_id` (fixed "C001"), and passes any existing `parameter` object.  
    - Configuration: Hardcoded values for demo/testing; in production, these can be dynamic expressions.  
    - Inputs: Trigger output from chat message node.  
    - Outputs: Metadata JSON for AI processing.  
    - Failure modes: Expression evaluation errors if input JSON is missing expected keys.  
    - Version: 3.4.

---

#### 1.2 AI Processing

- **Overview:**  
  This block fetches the draft content from Google Sheets, buffers short-term memory for context, runs the AI agent to enhance and format the content (including SEO metadata), and enforces JSON output schema.

- **Nodes Involved:**  
  - Fetch Optimized Draft from Sheets  
  - Short-Term Memory  
  - OpenAI Chat Model  
  - AI Agent (Publisher)  
  - Output Parser (JSON Enforcement)

- **Node Details:**

  - **Fetch Optimized Draft from Sheets**  
    - Type: Google Sheets Tool  
    - Role: Retrieves the draft content from a specific sheet and document ID.  
    - Configuration: Reads from sheet `"content_versions"` in document ID `"1fsnnXsU1n-iX-MEpLuw3XC6wHD-ek6OlkQe31ousk84"`. OAuth2 credentials linked to "automations@techdome.ai".  
    - Inputs: None directly; triggered by AI Agent node's tool input.  
    - Outputs: JSON containing draft content and context.  
    - Failure modes: Authentication errors, sheet not found, network issues.  
    - Version: 4.6.

  - **Short-Term Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains context over a session key `"publish-session"` to provide previous conversational or content context to the AI agent.  
    - Configuration: Uses a custom session key for context isolation.  
    - Inputs: Data from draft fetch and trigger.  
    - Outputs: Context data to AI Agent.  
    - Failure modes: Memory overflow, context loss, misconfiguration of session keys.  
    - Version: 1.3.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model (GPT-4.1 Mini)  
    - Role: Provides the underlying GPT-4 engine for the AI agent.  
    - Configuration: Model set to `"gpt-4.1-mini"`. Uses OpenAI credential with API key.  
    - Inputs: Prompt from AI Agent node.  
    - Outputs: AI-generated text response.  
    - Failure modes: API key invalid/expired, rate limits, model unavailability.  
    - Version: 1.2.

  - **AI Agent (Publisher)**  
    - Type: Langchain Agent  
    - Role: Core AI processing node that takes metadata, draft content, and context to generate fully formatted, SEO-optimized publishing data in structured JSON.  
    - Configuration:  
      - System message instructs the agent to prepare the article, format as HTML if for WordPress, and generate metadata (title, description, tags).  
      - Outputs strictly structured JSON.  
      - Uses expressions to inject topic, intent, content ID, platform, and context.  
    - Inputs: Metadata from Prepare Publishing Metadata, draft content from Sheets, context from memory, and OpenAI model for AI generation.  
    - Outputs: JSON with keys like `publish_data` containing all publishing fields.  
    - Failure modes: AI output not matching JSON schema, malformed JSON, API failures.  
    - Version: 2.1.

  - **Output Parser (JSON Enforcement)**  
    - Type: Langchain Output Parser Structured  
    - Role: Validates and enforces the AI output structure against a JSON schema example to ensure downstream compatibility.  
    - Configuration: JSON schema example provided with keys: `content_id`, `version_id`, `title`, `meta_description`, `tags`, `html_body`, `status`, `timestamp`.  
    - Inputs: AI Agent output.  
    - Outputs: Clean, validated JSON data.  
    - Failure modes: Parser rejects output, stopping workflow; schema mismatch.  
    - Version: 1.3.

---

#### 1.3 Storage

- **Overview:**  
  This block records the AI-enhanced and formatted content back into Google Sheets for version tracking and audit.

- **Nodes Involved:**  
  - Save Published Content to Sheets

- **Node Details:**

  - **Save Published Content to Sheets**  
    - Type: Google Sheets node  
    - Role: Appends or updates rows in the `"content_versions"` sheet with new publishing data from the AI agent.  
    - Configuration:  
      - Writes fields like HTML body, status, keywords, meta description, timestamps, content and version IDs, and meta title.  
      - Matches rows based on `content_id` for update or append.  
      - Uses the same Google Sheets OAuth2 credentials.  
    - Inputs: Validated JSON data from AI processing block.  
    - Outputs: Confirmation of write operation.  
    - Failure modes: Authentication failure, sheet lock, write conflicts.  
    - Version: 4.6.

---

#### 1.4 Approval Process

- **Overview:**  
  This block handles human approval by sending an email preview, checking approval status, publishing to a recipient email upon approval, and sending a Slack notification on success.

- **Nodes Involved:**  
  - Send Content for Approval  
  - Check Approval Status  
  - Publish to Recipient  
  - Send Success Notification to Slack

- **Node Details:**

  - **Send Content for Approval**  
    - Type: Gmail node  
    - Role: Sends an email containing the formatted HTML content for review and approval.  
    - Configuration:  
      - Sends to a placeholder email `[your-email@example.com]` (to be replaced).  
      - Email subject and body populated from AI output metadata and HTML body.  
      - Uses Gmail OAuth2 credentials for sending and awaiting reply (sendAndWait operation).  
    - Inputs: Published content JSON from AI agent.  
    - Outputs: Approval response data.  
    - Failure modes: Email delivery failures, OAuth issues, timeout waiting for reply.  
    - Version: 2.1.

  - **Check Approval Status**  
    - Type: If node  
    - Role: Evaluates the approval response JSON field `data.approved` for boolean `true`.  
    - Configuration:  
      - Condition checks if `{{ $json.data.approved }}` equals `true`.  
    - Inputs: Approval response from Gmail node.  
    - Outputs: Branches workflow to success or failure paths.  
    - Failure modes: Missing or malformed approval data, false negatives.  
    - Version: 2.2.

  - **Publish to Recipient**  
    - Type: Gmail node  
    - Role: Sends the final approved content to the actual publishing recipient.  
    - Configuration:  
      - Sends to placeholder `[recipient-email@example.com]` (to be replaced).  
      - Uses published content HTML and title for email body and subject.  
      - Gmail OAuth2 credentials distinct from approval email node.  
    - Inputs: Approved content from Check Approval Status node.  
    - Outputs: Confirmation of publishing email sent.  
    - Failure modes: Email send failure, OAuth issues.  
    - Version: 2.1.

  - **Send Success Notification to Slack**  
    - Type: Slack node  
    - Role: Posts a notification message to a Slack channel to confirm successful publication.  
    - Configuration:  
      - Message text includes published article title.  
      - Posts to channel ID `"C09GNB90TED"` (general-information).  
      - Uses Slack API token credentials.  
    - Inputs: Branch from approval success path.  
    - Outputs: Slack posting confirmation.  
    - Failure modes: Slack API token invalid, network issues.  
    - Version: 2.3.

---

#### 1.5 Security & Setup Notes

- **Overview:**  
  Provides important setup and credential information necessary for workflow deployment and operation.

- **Nodes Involved:**  
  - Overview (sticky note)  
  - Trigger Section (sticky note)  
  - AI Processing (sticky note)  
  - Storage (sticky note)  
  - Approval Process (sticky note)  
  - Security Note (sticky note)

- **Node Details:**  
  These are sticky notes that document:

  - Workflow purpose and high-level steps.  
  - Trigger initiation context.  
  - AI processing goals and outputs.  
  - Data storage rationale.  
  - Approval and notification flow.  
  - Credentials required:

    - Google Sheets OAuth2 for automation account.  
    - OpenAI API key for GPT-4 mini.  
    - Gmail OAuth2 for sending approval and publishing emails.  
    - Slack API token for notifications.

  - Reminder to replace credential IDs with own before deployment.

---

### 3. Summary Table

| Node Name                     | Node Type                                 | Functional Role                                    | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                                              |
|-------------------------------|------------------------------------------|---------------------------------------------------|-------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Overview                      | Sticky Note                              | Workflow purpose and setup instructions           | None                          | None                              | ## 📝 AI-Powered Content Publishing System\n\nSetup steps including credentials and spreadsheet configuration.                        |
| Trigger Section               | Sticky Note                              | Entry point explanation                            | None                          | None                              | ## 🎯 Workflow Entry Point\nChat trigger initiates publishing process with metadata prep.                                               |
| AI Processing                 | Sticky Note                              | AI content enhancement and formatting overview    | None                          | None                              | ## 🤖 AI Publishing Engine\nFetch drafts, format HTML, generate SEO metadata, output JSON.                                              |
| Storage                      | Sticky Note                              | Data storage and logging overview                  | None                          | None                              | ## 💾 Data Storage & Logging\nWrites content and metadata back to Sheets for tracking.                                                  |
| Approval Process             | Sticky Note                              | Approval and notification flow overview            | None                          | None                              | ## ✅ Approval & Notification Flow\nEmail approval, publish on approval, notify Slack.                                                   |
| Security Note                | Sticky Note                              | Credentials and security notes                      | None                          | None                              | ## 🔐 Credentials Required\nLists all required credentials and reminds to replace them before deployment.                              |
| When chat message received    | Langchain Chat Trigger                   | Workflow trigger on chat message                    | None                          | Prepare Publishing Metadata        |                                                                                                                                          |
| Prepare Publishing Metadata   | Set                                     | Prepares initial metadata for AI processing        | When chat message received    | AI Agent (Publisher)               |                                                                                                                                          |
| Fetch Optimized Draft from Sheets | Google Sheets Tool                      | Reads draft content from Google Sheets             | AI Agent (Publisher) (tool)   | AI Agent (Publisher) (tool)        |                                                                                                                                          |
| Short-Term Memory            | Langchain Memory Buffer Window           | Maintains session context for AI                    | AI Agent (Publisher) (memory) | AI Agent (Publisher) (memory)      |                                                                                                                                          |
| OpenAI Chat Model            | Langchain OpenAI Chat Model               | Provides GPT-4 model for AI agent                   | AI Agent (Publisher)          | AI Agent (Publisher)               |                                                                                                                                          |
| AI Agent (Publisher)          | Langchain Agent                          | Core AI logic for content formatting and metadata  | Prepare Publishing Metadata, Fetch Optimized Draft, Short-Term Memory, OpenAI Chat Model | Save Published Content to Sheets, Send Content for Approval |                                                                                                                                          |
| Output Parser (JSON Enforcement) | Langchain Output Parser Structured      | Enforces AI output JSON schema                       | AI Agent (Publisher) (outputParser) | AI Agent (Publisher)               |                                                                                                                                          |
| Save Published Content to Sheets | Google Sheets                          | Stores formatted content and metadata back to Sheets | AI Agent (Publisher)          | Send Content for Approval          |                                                                                                                                          |
| Send Content for Approval     | Gmail                                   | Sends email for human approval of content           | Save Published Content to Sheets | Check Approval Status              |                                                                                                                                          |
| Check Approval Status         | If                                      | Checks if content was approved                       | Send Content for Approval     | Publish to Recipient, Send Success Notification to Slack |                                                                                                                                          |
| Publish to Recipient          | Gmail                                   | Sends final published content email                  | Check Approval Status         | None                             |                                                                                                                                          |
| Send Success Notification to Slack | Slack                                | Sends Slack notification on successful publishing  | Check Approval Status         | None                             |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Langchain Chat Trigger** node named `When chat message received`.  
   - Configure with default webhook settings to receive chat messages.

2. **Add Metadata Preparation:**  
   - Add a **Set** node named `Prepare Publishing Metadata`.  
   - Set fields:  
     - `intent` = `"publish"`  
     - `topic` = `"AI Seo Basics"`  
     - `content_id` = `"C001"`  
     - `parameter` = `{{ $json.parameter }}` (pass through any platform parameters)  
   - Connect `When chat message received` → `Prepare Publishing Metadata`.

3. **Add AI Agent Components:**  
   - Add a **Google Sheets Tool** node named `Fetch Optimized Draft from Sheets`.  
     - Configure with OAuth2 credentials for Google Sheets.  
     - Set Spreadsheet ID: `1fsnnXsU1n-iX-MEpLuw3XC6wHD-ek6OlkQe31ousk84`  
     - Sheet Name: `content_versions`  
   - Add a **Langchain Memory Buffer Window** node named `Short-Term Memory`.  
     - Set session key: `"publish-session"`, session ID type: custom key.  
   - Add **Langchain OpenAI Chat Model** node named `OpenAI Chat Model`.  
     - Choose model: `gpt-4.1-mini`  
     - Attach OpenAI API credentials.  
   - Add **Langchain Agent** node named `AI Agent (Publisher)`.  
     - System Message:  
       ```
       You are "Publisher Agent", an AI responsible for preparing and posting optimized articles.

       Your tasks:
       1. Take the optimized draft and prepare it for publishing.
       2. Clean and format for web standards (HTML-ready headings, tags, meta info).
       3. Generate metadata, tags, and keywords.
       4. Always return structured JSON only, in this format:
       {
         "publish_data": {
           "content_id": "string",
           "version_id": "string",
           "title": "string",
           "meta_description": "string",
           "tags": ["string"],
           "html_body": "string",
           "status": "Published",
           "timestamp": "number"
         }
       }
       ```
     - Prompt (user message):  
       ```
       Topic: {{ $json.topic }}
       Intent: {{ $json.intent }}
       Content ID: {{ $json.content_id }}
       Platform: {{ $json.parameter.platform || 'Internal CMS' }}

       Context (from Sheets or memory): {{ $json.context || $memory || 'No context found' }}

       Prepare the article for publishing. Ensure metadata, title, tags, and description are optimized. If the platform is WordPress, format the article in valid HTML. Return only structured JSON in the required format.
       ```
     - Connect inputs:  
       - Metadata from `Prepare Publishing Metadata`  
       - Tool input to `Fetch Optimized Draft from Sheets`  
       - Memory input to `Short-Term Memory`  
       - AI model input to `OpenAI Chat Model`  
   - Add **Langchain Output Parser Structured** node named `Output Parser (JSON Enforcement)`.  
     - Configure with JSON schema example matching `publish_data` fields.  
     - Connect output of this parser back to the AI Agent node’s output parser input.

   - Connect nodes:  
     `Prepare Publishing Metadata` → `AI Agent (Publisher)`  
     `Fetch Optimized Draft from Sheets` (tool) → `AI Agent (Publisher)`  
     `Short-Term Memory` (memory) → `AI Agent (Publisher)`  
     `OpenAI Chat Model` (language model) → `AI Agent (Publisher)`  
     `Output Parser (JSON Enforcement)` → `AI Agent (Publisher)` (outputParser)

4. **Add Storage Node:**  
   - Add **Google Sheets** node named `Save Published Content to Sheets`.  
   - Configure with same Google Sheets OAuth2 credentials.  
   - Use Spreadsheet ID and Sheet Name as above.  
   - Mapping mode: Define fields to write: `html`, `status`, `keywords`, `meta_desc`, `timestamp`, `content_id`, `meta_title`, `version_id`.  
   - Matching Column: `content_id` to append or update.  
   - Connect AI Agent (Publisher) main output to this node.

5. **Add Approval Email Node:**  
   - Add **Gmail** node named `Send Content for Approval`.  
   - Configure with Gmail OAuth2 credentials for approval emails.  
   - Set recipient email (replace `[your-email@example.com]` with real address).  
   - Subject: `={{ $json.output.publish_data.title }}`  
   - Message body: `={{ $json.output.publish_data.html_body }}`  
   - Operation: `sendAndWait` to await reply.  
   - Connect `Save Published Content to Sheets` → `Send Content for Approval`.

6. **Add Approval Check Node:**  
   - Add **If** node named `Check Approval Status`.  
   - Condition: Check if `{{ $json.data.approved }}` equals `true` (Boolean).  
   - Connect `Send Content for Approval` → `Check Approval Status`.

7. **Add Publishing Email Node:**  
   - Add **Gmail** node named `Publish to Recipient`.  
   - Configure with separate Gmail OAuth2 credentials if desired.  
   - Set recipient email (replace `[recipient-email@example.com]` with real address).  
   - Subject and message body sourced from the AI Agent output JSON.  
   - Connect `Check Approval Status` (true path) → `Publish to Recipient`.

8. **Add Slack Notification Node:**  
   - Add **Slack** node named `Send Success Notification to Slack`.  
   - Configure with Slack API token credentials.  
   - Set channel ID (e.g., `C09GNB90TED` for general-information).  
   - Message: `"Your {{ $('AI Agent (Publisher)').item.json.output.publish_data.title }} has been Successfully Published."`  
   - Connect `Check Approval Status` (true path) → `Send Success Notification to Slack`.

9. **Finish Connections:**  
   - Connect `Publish to Recipient` and `Send Success Notification to Slack` outputs to end workflow.  
   - Ensure all credentials are configured and valid.  
   - Replace all placeholder emails and IDs with actual production values.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Credentials required: Google Sheets OAuth2, OpenAI API key for GPT-4 mini, Gmail OAuth2, Slack API token.      | Security Note sticky note in workflow.                                                            |
| Replace all credential IDs and email placeholders with your own before deployment.                             | Security Note sticky note.                                                                         |
| Workflow automates final stage of content creation from Sheets to CMS or email with AI formatting and SEO.     | Overview sticky note.                                                                              |
| Chat trigger starts the process, useful for integrating with conversational UIs or chatbots.                   | Trigger Section sticky note.                                                                       |
| AI Agent enforces JSON output schema for consistent downstream processing.                                     | AI Processing sticky note.                                                                         |
| Approval email uses Gmail sendAndWait operation to pause workflow until human response.                        | Approval Process sticky note.                                                                      |
| Slack notification channel: `C09GNB90TED` (general-information)                                                | Slack node configuration.                                                                         |
| Google Sheets doc ID: `1fsnnXsU1n-iX-MEpLuw3XC6wHD-ek6OlkQe31ousk84`, sheet name: `content_versions`.         | Google Sheets node configuration.                                                                 |
| OpenAI GPT-4 mini model selected for balanced performance and cost.                                           | OpenAI Chat Model node configuration.                                                             |

---

This document captures all nodes and their logical roles in the workflow "Streamline Content Publishing from Google Sheets to Email & Slack with AI Formatting." It enables advanced users and automation agents to understand, reproduce, and modify the workflow confidently.