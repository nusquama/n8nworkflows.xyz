Gmail AI Email Manager

https://n8nworkflows.xyz/workflows/gmail-ai-email-manager-4722


# Gmail AI Email Manager

---
### 1. Workflow Overview

This workflow, titled **Gmail AI Email Manager**, is designed to automatically classify incoming Gmail messages for a specific user (`max@trigify.io`) using AI-driven analysis. Its main purpose is to streamline email management by categorizing emails into appropriate Gmail labels based on content, sender history, and metadata. This helps prioritize responses, filter marketing, notifications, and other categories, enhancing productivity and email organization.

**Target Use Cases:**
- Automated classification of incoming emails by applying intelligent labels.
- Differentiation between cold emails, marketing, notifications, FYI, and actionable emails.
- Contextual understanding of email sender history to improve labeling accuracy.
- Integration with Gmail for label application in real-time.

**Logical Blocks:**

- **1.1 Input Reception and Email Retrieval:**  
  Captures newly received Gmail emails and fetches full email data.

- **1.2 AI-Powered Email Classification:**  
  Uses an AI agent to classify the email based on email content, headers, and prior email history.

- **1.3 Prior Email History Lookup:**  
  Queries Gmail inbox and sent folder to determine if prior correspondence exists with the sender.

- **1.4 Output Parsing and Label Application:**  
  Parses AI output to extract a Gmail label ID and applies the corresponding label to the email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Email Retrieval

- **Overview:**  
  This block listens for new incoming emails in Gmail and retrieves full email details for processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Gmail

- **Node Details:**

  - **Gmail Trigger:**  
    - *Type & Role:* Trigger node that listens for new Gmail messages.  
    - *Configuration:* Polls every minute with no additional filters, using OAuth2 credentials for the Gmail account.  
    - *Input/Output:* No input; outputs metadata of incoming emails (message IDs, basic headers).  
    - *Edge Cases:* Gmail API quota limits, OAuth token expiration, network timeouts.

  - **Gmail:**  
    - *Type & Role:* Retrieves full email data for the message ID received from the trigger.  
    - *Configuration:* Operation "get" by message ID, with full message details (not simple mode). OAuth2 credentials used.  
    - *Input:* Message ID from Gmail Trigger.  
    - *Output:* Complete email JSON data including headers, body, labels, etc.  
    - *Edge Cases:* Message deletion after trigger, API errors, large message payload handling.

---

#### 2.2 AI-Powered Email Classification

- **Overview:**  
  This block uses an Anthropic-based AI chat model wrapped in an AI Agent node to analyze the email and classify it into a label based on a complex system of heuristics and context.

- **Nodes Involved:**  
  - Anthropic Chat Model  
  - AI Agent  
  - Structured Output Parser

- **Node Details:**

  - **Anthropic Chat Model:**  
    - *Type & Role:* Language Model node for AI text generation using Anthropic Claude Sonnet 4.  
    - *Configuration:* Model set to "claude-sonnet-4-20250514", no additional options. Uses Anthropic API credentials.  
    - *Input:* Prompt text from AI Agent node.  
    - *Output:* AI-generated classification response.  
    - *Edge Cases:* API rate limits, network errors, model response latency.

  - **AI Agent:**  
    - *Type & Role:* Orchestrates AI workflow; sends prompt to Anthropic chat model and handles tools for prior history checking.  
    - *Configuration:*  
      - System message instructs classification rules referencing prior email history and email metadata.  
      - Uses two AI tools for querying email history (Get Email, Check Sent).  
      - Output parser node attached to parse AI JSON output.  
    - *Input:* Full email JSON from Gmail node.  
    - *Output:* Label ID string to be used for Gmail labeling.  
    - *Edge Cases:* Expression evaluation errors, incorrect variable interpolation, AI prompt ambiguity, API errors.

  - **Structured Output Parser:**  
    - *Type & Role:* Parses AI output into a structured JSON format specifying label and label ID.  
    - *Configuration:* JSON schema example expects keys `"label"` and `"label ID"`.  
    - *Input:* AI Agent output text.  
    - *Output:* Structured JSON for downstream use.  
    - *Edge Cases:* Malformed AI responses, parsing failures.

---

#### 2.3 Prior Email History Lookup

- **Overview:**  
  This block queries Gmail to check if there is prior email history between the user and the sender, querying both inbox and sent folders.

- **Nodes Involved:**  
  - Get Email  
  - Check Sent

- **Node Details:**

  - **Get Email:**  
    - *Type & Role:* Gmail Tool node to search inbox for emails from the sender address.  
    - *Configuration:* Query filter `from:<sender_email>`. Returns all matching emails (`returnAll` boolean controlled via AI Agent). Uses separate Gmail OAuth2 credential.  
    - *Input:* Sender email dynamically set via AI Agent context.  
    - *Output:* List of emails matching the query.  
    - *Edge Cases:* Gmail API quota, sender email format issues, empty results.

  - **Check Sent:**  
    - *Type & Role:* Gmail Tool node to search sent folder for emails to the sender address.  
    - *Configuration:* Query filter `to:<sender_email>` with label ID "SENT". Returns all matching emails. Uses main Gmail OAuth2 credential.  
    - *Input:* Sender email from AI Agent context.  
    - *Output:* List of sent emails matching criteria.  
    - *Edge Cases:* API limits, no sent emails found, label filtering errors.

---

#### 2.4 Output Parsing and Label Application

- **Overview:**  
  Uses the classification label ID obtained from the AI Agent's output to apply the corresponding Gmail label to the original email.

- **Nodes Involved:**  
  - Gmail1

- **Node Details:**

  - **Gmail1:**  
    - *Type & Role:* Gmail node to add label(s) to the email.  
    - *Configuration:* Operation "addLabels" with label ID extracted from AI Agent output via structured parser. Uses OAuth2 credentials.  
    - *Input:* Label ID from AI Agent and message ID from Gmail node.  
    - *Output:* Confirmation of label addition.  
    - *Edge Cases:* Invalid label IDs, label not existing in Gmail, API errors, email message deletion before labeling.

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                         | Input Node(s)          | Output Node(s)          | Sticky Note                                           |
|-------------------------|---------------------------------------------|---------------------------------------|-----------------------|-------------------------|-------------------------------------------------------|
| Gmail Trigger           | n8n-nodes-base.gmailTrigger                  | Trigger new incoming Gmail emails     | None                  | Gmail                   |                                                       |
| Gmail                   | n8n-nodes-base.gmail                         | Retrieve full email data               | Gmail Trigger         | AI Agent                |                                                       |
| Anthropic Chat Model    | @n8n/n8n-nodes-langchain.lmChatAnthropic    | AI language model for classification  | AI Agent (ai_languageModel) | AI Agent            |                                                       |
| AI Agent               | @n8n/n8n-nodes-langchain.agent               | Orchestrate AI prompt and tools       | Gmail, Get Email, Check Sent | Gmail1              |                                                       |
| Structured Output Parser| @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI output to JSON                | AI Agent (ai_outputParser) | AI Agent              |                                                       |
| Get Email              | n8n-nodes-base.gmailTool                      | Search inbox for prior emails from sender | AI Agent (ai_tool)   | AI Agent                |                                                       |
| Check Sent             | n8n-nodes-base.gmailTool                      | Search sent folder for emails to sender | AI Agent (ai_tool)    | AI Agent                |                                                       |
| Gmail1                 | n8n-nodes-base.gmail                         | Apply Gmail label to email             | AI Agent               | None                    |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Gmail Trigger node:**  
   - Type: `Gmail Trigger`  
   - Set to poll every minute (no filters).  
   - Authenticate with Gmail OAuth2 credentials for the user account (`max@trigify.io`).

2. **Add a Gmail node to retrieve full email details:**  
   - Operation: `get` by Message ID.  
   - Use the message ID from the Gmail Trigger node (`{{$json["id"]}}`).  
   - Use the same Gmail OAuth2 credentials.

3. **Add an Anthropic Chat Model node:**  
   - Model: `claude-sonnet-4-20250514`.  
   - No extra options.  
   - Authenticate with Anthropic API credentials.

4. **Add AI Agent node:**  
   - Set prompt mode to define.  
   - Paste the detailed system message and prompt that includes instructions for classification based on email data and prior email history.  
   - Configure AI Agent to use:  
     - Anthropic Chat Model node as the language model.  
     - Get Email and Check Sent nodes as AI tools for querying prior correspondence.  
     - Structured Output Parser node as the output parser.  
   - Pass email JSON from the Gmail node into the prompt parameters with expressions for sender, recipients, subject, body, headers, labels, etc.

5. **Add Structured Output Parser node:**  
   - Set JSON schema example with keys `label` and `label ID`.  
   - Connect AI Agent output to this parser.

6. **Create Get Email node:**  
   - Use Gmail Tool node.  
   - Operation: `getAll`.  
   - Query: `from:{{ $fromAI('email') }}` to find all emails from the sender.  
   - Return all results.  
   - Authenticate with a Gmail OAuth2 credential (can be the same or different).  
   - Connect as AI tool input to AI Agent.

7. **Create Check Sent node:**  
   - Use Gmail Tool node.  
   - Operation: `getAll`.  
   - Query: `to:{{ $fromAI('email') }}` with label filter "SENT".  
   - Authenticate with Gmail OAuth2 credentials.  
   - Connect as AI tool input to AI Agent.

8. **Add Gmail node (Gmail1) to apply labels:**  
   - Operation: `addLabels`.  
   - Label IDs: Use expression to get label ID from AI Agent structured output (`{{$json.output['label ID']}}`).  
   - Message ID: Use current email ID from Gmail node.  
   - Use Gmail OAuth2 credentials.  
   - Connect AI Agent main output to this node.

9. **Connect nodes according to dependencies:**  
   - Gmail Trigger → Gmail  
   - Gmail → AI Agent  
   - AI Agent → Gmail1  
   - AI Agent → Anthropic Chat Model (language model)  
   - AI Agent → Structured Output Parser (output parser)  
   - AI Agent → Get Email (AI tool)  
   - AI Agent → Check Sent (AI tool)

10. **Set credentials properly:**  
    - Gmail OAuth2 credentials for all Gmail nodes.  
    - Anthropic API credentials for Anthropic Chat Model node.

11. **Activate workflow and test with incoming emails.**

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow uses Anthropic Claude Sonnet 4 model for AI classification.                                                      | Anthropic API reference: https://www.anthropic.com/index/api                                         |
| Gmail OAuth2 credentials must have full read/write access to labels and messages for correct operation.                   | Gmail API scopes: https://developers.google.com/gmail/api/auth/scopes                               |
| Classification prompt includes detailed logic for distinguishing Marketing, Notifications, FYI, To Respond, Comment, etc.| Embedded in AI Agent system message for maintainability and tuning.                                 |
| Prior email history checking combines inbox and sent folder searches for comprehensive context.                          | Uses Gmail Tool nodes with dynamic query filters based on sender email from AI Agent.                |
| Structured Output Parser expects JSON with keys `"label"` and `"label ID"` for downstream label application.             | Ensures AI output can be reliably parsed and used for Gmail label assignment.                        |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. The process strictly adheres to content policies and contains no illegal or offensive elements. All data processed are legal and public.