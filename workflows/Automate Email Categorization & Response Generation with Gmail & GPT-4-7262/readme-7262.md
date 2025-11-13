Automate Email Categorization & Response Generation with Gmail & GPT-4

https://n8nworkflows.xyz/workflows/automate-email-categorization---response-generation-with-gmail---gpt-4-7262


# Automate Email Categorization & Response Generation with Gmail & GPT-4

### 1. Workflow Overview

This workflow automates the categorization and response generation process for incoming Gmail emails using AI analysis (GPT-4). It targets users who want to streamline email management by automatically sorting emails, detecting spam, generating reply drafts, notifying for high-priority messages, and logging interactions for audit or review.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming Gmail emails via webhook and fetches detailed email content.
- **1.2 AI Processing:** Analyzes email content using OpenAI GPT-4 to classify, prioritize, and suggest actions.
- **1.3 Routing & Decision Making:** Routes emails based on priority and category (e.g., spam detection, reply needed).
- **1.4 Action Execution:** Executes corresponding actions such as sending alerts, creating reply drafts, moving spam, categorizing emails with labels, and logging.
- **1.5 Finalization:** Sends a webhook response confirming processing completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new Gmail push notifications, triggered through a webhook, and retrieves full email details for further processing.

**Nodes Involved:**  
- 📧 Gmail Webhook Trigger  
- 📩 Get Email Details

**Node Details:**

- **📧 Gmail Webhook Trigger**  
  - Type: Webhook Trigger  
  - Role: Entry point triggered by Gmail push notification (HTTP POST)  
  - Configuration:  
    - Path set to `gmail-webhook`  
    - HTTP method POST, responds with data node output  
  - Inputs: External Gmail push notifications  
  - Outputs: JSON containing minimal email data including messageId  
  - Setup Notes: Requires Google Cloud Console Gmail webhook configuration and push notification setup  
  - Edge Cases: Webhook URL misconfiguration, Gmail API push notification delays, or failures

- **📩 Get Email Details**  
  - Type: Gmail Node (OAuth2)  
  - Role: Retrieves full email data using messageId from webhook payload  
  - Configuration:  
    - Operation: Get email by ID  
    - Uses OAuth2 credentials with scopes `gmail.readonly` and `gmail.modify`  
  - Inputs: messageId from webhook node  
  - Outputs: Full email JSON payload including headers, snippet, and IDs  
  - Edge Cases: OAuth token expiration, permission errors, invalid messageId

---

#### 2.2 AI Processing

**Overview:**  
Analyzes the email with OpenAI GPT-4 to classify it, detect sentiment and priority, recommend actions, and suggest draft replies.

**Nodes Involved:**  
- 🤖 AI Email Analyzer

**Node Details:**

- **🤖 AI Email Analyzer**  
  - Type: OpenAI Node  
  - Role: Uses GPT-4 model for deep email content analysis and structured JSON output  
  - Configuration:  
    - Model: gpt-4  
    - Prompt: Custom prompt requesting JSON with fields like category, sentiment, priority, action_needed, summary, suggested_response, keywords  
    - Temperature: 0.3 (to ensure consistency)  
    - Inputs: Email headers (From, Subject) and snippet from previous node  
  - Outputs: AI-generated JSON content inside `choices[0].message.content`  
  - Edge Cases: API rate limits, malformed responses, prompt failures, network issues

---

#### 2.3 Routing & Decision Making

**Overview:**  
Routes the email flow based on AI-assigned priority and category for subsequent actions.

**Nodes Involved:**  
- 🚦 Priority Router  
- 🛡️ Spam Detector  
- 💬 Auto-Reply Detector

**Node Details:**

- **🚦 Priority Router**  
  - Type: IF Node  
  - Role: Routes emails if priority is "high" for immediate alert, else continues to spam detection  
  - Conditions: `priority === "high"` parsed from AI response JSON  
  - Outputs: Two branches - high priority or other  
  - Edge Cases: Parsing errors if AI JSON is invalid

- **🛡️ Spam Detector**  
  - Type: IF Node  
  - Role: Detects spam emails based on AI category and routes to spam handling or reply detection  
  - Conditions: `category === "spam"` parsed from AI response JSON  
  - Outputs: Two branches - spam or non-spam  
  - Edge Cases: Misclassification by AI, parsing failures

- **💬 Auto-Reply Detector**  
  - Type: IF Node  
  - Role: Detects if the AI suggests a reply is needed to the email  
  - Conditions: `action_needed === "reply"` parsed from AI response JSON  
  - Outputs: Two branches - reply needed or not  
  - Edge Cases: Incorrect action detection

---

#### 2.4 Action Execution

**Overview:**  
Performs actions such as alerting, drafting replies, moving spam, labeling emails, and logging details.

**Nodes Involved:**  
- 🚨 High Priority Alert  
- 🗑️ Move to Spam  
- 📝 Create Reply Draft  
- 🏷️ Categorize Email  
- 📊 Log to Spreadsheet

**Node Details:**

- **🚨 High Priority Alert**  
  - Type: Gmail Node  
  - Role: Sends notification email for high priority messages  
  - Configuration:  
    - Custom alert email recipient (must update)  
    - Subject: "🚨 High Priority Email Alert"  
    - Message body includes sender, subject, AI summary, suggested action, and Gmail link  
  - Inputs: Email details and AI analysis  
  - Edge Cases: Email sending failure, incorrect recipient

- **🗑️ Move to Spam**  
  - Type: Gmail Node  
  - Role: Moves detected spam emails to the Spam folder, removing from inbox and applying spam label  
  - Configuration: Modify operation with appropriate Gmail labels  
  - Edge Cases: Label misconfiguration, permission issues

- **📝 Create Reply Draft**  
  - Type: Gmail Node  
  - Role: Creates a draft reply using AI suggested response  
  - Configuration:  
    - Draft subject prepended with "Re:" and original subject  
    - Message includes AI reply with note to review before sending  
  - Edge Cases: Draft creation failures, invalid message content

- **🏷️ Categorize Email**  
  - Type: Gmail Node  
  - Role: Applies Gmail labels based on AI classification (URGENT, IMPORTANT, PROMOTIONAL, PERSONAL, WORK, etc.)  
  - Configuration: Modify operation to add appropriate labels  
  - Edge Cases: Missing labels in Gmail, permission issues

- **📊 Log to Spreadsheet**  
  - Type: Google Sheets Node  
  - Role: Logs email metadata, category, priority, sentiment, AI summary, and action taken into a Google Sheet for record keeping  
  - Configuration: Append or update operation with spreadsheet and sheet ID (must update)  
  - Edge Cases: Google API permission issues, sheet sharing misconfiguration

---

#### 2.5 Finalization

**Overview:**  
Sends a JSON response back to Gmail confirming processing status and AI classification, required for webhook completion.

**Nodes Involved:**  
- ✅ Webhook Response

**Node Details:**

- **✅ Webhook Response**  
  - Type: Respond to Webhook Node  
  - Role: Returns a JSON object confirming successful processing and includes key AI analysis results (category, priority, action taken, etc.)  
  - Outputs: HTTP 200 response to Gmail webhook sender  
  - Edge Cases: Response timeout, malformed JSON

---

### 3. Summary Table

| Node Name               | Node Type                | Functional Role                         | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                                         |
|-------------------------|--------------------------|---------------------------------------|------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------|
| 📧 Gmail Webhook Trigger | Webhook Trigger          | Entry point for Gmail push notifications | None                   | 📩 Get Email Details              | 🔧 SETUP REQUIRED: 1. Copy webhook URL after saving 2. Configure Gmail webhook in Google Cloud Console 3. Set up Gmail push notifications 4. Test with sample email |
| 📩 Get Email Details     | Gmail Node               | Fetch full email content               | 📧 Gmail Webhook Trigger | 🤖 AI Email Analyzer             | 🔧 SETUP REQUIRED: 1. Create Gmail OAuth2 credentials 2. Add scopes: gmail.readonly, gmail.modify 3. Authorize connection 4. Test connection |
| 🤖 AI Email Analyzer     | OpenAI GPT-4 Node        | Analyze email content with AI          | 📩 Get Email Details     | 🚦 Priority Router               | 🔧 SETUP REQUIRED: 1. Get OpenAI API key 2. Add to credentials 3. Choose model (gpt-4 recommended) 4. Adjust temperature 5. Test with sample emails |
| 🚦 Priority Router       | IF Node                  | Routes emails based on priority        | 🤖 AI Email Analyzer     | 🚨 High Priority Alert, 🛡️ Spam Detector | 📝 POST-IT NOTE: Routes emails based on AI analysis - High priority → Immediate notification - Medium/Low → Standard processing - Customize conditions as needed |
| 🚨 High Priority Alert   | Gmail Node               | Send alert email for high priority     | 🚦 Priority Router       | 🏷️ Categorize Email             | 🔧 SETUP REQUIRED: 1. Update notification email address 2. Customize alert message 3. Consider Slack/Teams 4. Test notification delivery |
| 🛡️ Spam Detector         | IF Node                  | Detect spam category                   | 🚦 Priority Router       | 🗑️ Move to Spam, 💬 Auto-Reply Detector | 📝 POST-IT NOTE: Detects spam emails - Auto-delete or move to spam - Log spam attempts - Update spam filters |
| 🗑️ Move to Spam           | Gmail Node               | Move emails to spam folder              | 🛡️ Spam Detector         | 📊 Log to Spreadsheet            | 📝 POST-IT NOTE: Moves spam to spam folder - Removes from inbox - Adds spam label - Consider auto-delete after time |
| 💬 Auto-Reply Detector    | IF Node                  | Detect if reply is needed               | 🛡️ Spam Detector         | 📝 Create Reply Draft, 🏷️ Categorize Email | 📝 POST-IT NOTE: Detects emails needing replies - AI suggests response - Option for auto-send or draft - Customize reply rules |
| 📝 Create Reply Draft     | Gmail Node               | Creates draft reply based on AI output | 💬 Auto-Reply Detector    | 🏷️ Categorize Email             | 📝 POST-IT NOTE: Creates AI-generated reply draft - Review before sending - Customize response style - Option to auto-send |
| 🏷️ Categorize Email       | Gmail Node               | Apply Gmail labels based on AI category | 🚨 High Priority Alert, 📝 Create Reply Draft, 💬 Auto-Reply Detector | 📊 Log to Spreadsheet            | 🔧 SETUP REQUIRED: 1. Create Gmail labels: URGENT, IMPORTANT, PROMOTIONAL, PERSONAL, WORK, PROJECTS 2. Customize categories 3. Set up label colors |
| 📊 Log to Spreadsheet     | Google Sheets Node       | Log email data and AI analysis          | 🏷️ Categorize Email, 🗑️ Move to Spam | ✅ Webhook Response            | 🔧 SETUP REQUIRED: 1. Create Google Sheets log 2. Add headers: timestamp, messageId, from, subject, category, priority, sentiment, action_taken, ai_summary 3. Share sheet with service account 4. Update spreadsheet ID |
| ✅ Webhook Response       | Respond to Webhook Node  | Sends confirmation response to Gmail   | 📊 Log to Spreadsheet    | None                            | 📝 POST-IT NOTE: Sends response back to Gmail - Confirms processing - Includes analysis results - Required for webhook completion |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Path: `gmail-webhook`  
   - HTTP Method: POST  
   - Response Mode: Response Node  
   - Save node and copy webhook URL for Gmail setup.

2. **Configure Gmail API Credentials in Google Cloud Console**  
   - Enable Gmail API  
   - Create OAuth 2.0 credentials  
   - Add redirect URI: `https://your-n8n-instance.com/rest/oauth2-credential/callback`  
   - Scopes: `gmail.readonly`, `gmail.modify`, `gmail.compose`  
   - In n8n, create Gmail OAuth2 credential with client ID, secret, and scopes.

3. **Add Gmail Node to Get Email Details**  
   - Type: Gmail  
   - Operation: Get (email by messageId)  
   - Use expression `={{ $json.message.data.messageId }}` to get messageId from webhook payload  
   - Attach Gmail OAuth2 credential.

4. **Add OpenAI Node for Email Analysis**  
   - Type: OpenAI  
   - Model: GPT-4  
   - Temperature: 0.3  
   - Prompt: Provide a JSON structure to classify email with fields category, sentiment, priority, action_needed, summary, suggested_response, keywords.  
   - Use expressions to pass From, Subject, and snippet from Gmail node.  
   - Attach OpenAI API credential.

5. **Add IF Node "Priority Router"**  
   - Condition: Check if priority (parsed from AI JSON) equals "high"  
   - Output 1 (True): High Priority Alert branch  
   - Output 2 (False): Spam Detector branch

6. **Add Gmail Node "High Priority Alert"**  
   - Operation: Send Email  
   - Recipient: Update with notification email address  
   - Subject: "🚨 High Priority Email Alert"  
   - Message: Include sender, subject, AI summary, suggested action, and Gmail link using expressions.

7. **Add IF Node "Spam Detector"**  
   - Condition: Check if category (from AI JSON) equals "spam"  
   - Output 1 (True): Move to Spam branch  
   - Output 2 (False): Auto-Reply Detector branch

8. **Add Gmail Node "Move to Spam"**  
   - Operation: Modify email labels  
   - Action: Remove from Inbox, add Spam label  
   - Use messageId from Gmail node.

9. **Add IF Node "Auto-Reply Detector"**  
   - Condition: Check if action_needed (from AI JSON) equals "reply"  
   - Output 1 (True): Create Reply Draft branch  
   - Output 2 (False): Categorize Email branch

10. **Add Gmail Node "Create Reply Draft"**  
    - Operation: Create draft email  
    - Subject: Prepend "Re:" to original subject  
    - Message: Use AI suggested response with a note to review before sending

11. **Add Gmail Node "Categorize Email"**  
    - Operation: Modify labels  
    - Apply Gmail labels such as URGENT, IMPORTANT, PROMOTIONAL, PERSONAL, WORK based on AI category  
    - Ensure labels exist in Gmail account.

12. **Add Google Sheets Node "Log to Spreadsheet"**  
    - Operation: Append or update spreadsheet row  
    - Spreadsheet and sheet ID: Set to your Google Sheet for logging  
    - Fields: timestamp, messageId, from, subject, category, priority, sentiment, action_taken, ai_summary  
    - Share sheet with service account email.

13. **Add Respond to Webhook Node**  
    - Respond with JSON including details (category, priority, messageId, action_taken) and success message.

14. **Connect nodes according to logical flow:**  
    - Gmail Webhook Trigger → Get Email Details → AI Email Analyzer → Priority Router  
    - Priority Router (high) → High Priority Alert → Categorize Email  
    - Priority Router (non-high) → Spam Detector  
    - Spam Detector (spam) → Move to Spam → Log to Spreadsheet → Webhook Response  
    - Spam Detector (not spam) → Auto-Reply Detector  
    - Auto-Reply Detector (reply) → Create Reply Draft → Categorize Email  
    - Auto-Reply Detector (no reply) → Categorize Email  
    - Categorize Email → Log to Spreadsheet → Webhook Response

15. **Test the workflow:**  
    - Save and activate  
    - Send test email to Gmail account  
    - Verify webhook is triggered, AI analysis is performed, emails are labeled, replies drafted, alerts sent, and logs recorded.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                              | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Need a tailor-made workflow? Tell me about your business and get a free proposal: **Start here → Custom Automation Form** https://taskmorphr.com/contact                                                                                                                                                 | Sticky Note9                                                                                           |
| Curious what automation could save you? Run the 60-second calculator: **ROI / Cost Comparison** https://taskmorphr.com/cost-comparison                                                                                                                                                                  | Sticky Note9                                                                                           |
| Reach me directly: `paul@taskmorphr.com`                                                                                                                                                                                                                                                                | Sticky Note9                                                                                           |
| Browse every ready-made workflow: **Full Template Pack — coming soon** https://n8n.io/creators/diagopl/                                                                                                                                                                                                  | Sticky Note10                                                                                          |
| Gmail AI Email Manager - Setup Guide with detailed steps for Gmail API, AI service setup, credentials, labels, and testing. Includes checklist and best practices for smooth setup and customization.                                                                                                      | Sticky Note (large setup guide node)                                                                   |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies and handling only legal and public data.