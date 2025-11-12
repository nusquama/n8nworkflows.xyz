Automated Email Classification & Response System with Gmail, GPT, and Sheets

https://n8nworkflows.xyz/workflows/automated-email-classification---response-system-with-gmail--gpt--and-sheets-7839


# Automated Email Classification & Response System with Gmail, GPT, and Sheets

---

### 1. Workflow Overview

This workflow automates the classification and response process for incoming Gmail emails using AI language models (OpenAI GPT), Gmail API, and Google Sheets for logging. It targets use cases where a business or support team needs to efficiently categorize emails into Support, Sales, Complaints, Information, or Other, automatically label them in Gmail, generate draft email replies for Support and Sales inquiries, and log all processed emails and errors for auditing.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Watches Gmail inbox for new unread emails.
- **1.2 Email Classification:** Uses an AI text classifier to categorize emails into predefined categories.
- **1.3 Decision Setting & Labeling:** Sets a decision label internally and applies corresponding Gmail labels based on classification.
- **1.4 AI Response Generation:** For Support and Sales categories, generates professional draft replies using OpenAI GPT models.
- **1.5 Draft Creation and Thread Labeling:** Creates Gmail drafts from AI-generated content and adds labels to email threads.
- **1.6 Logging & Error Handling:** Logs processed email details and decisions into Google Sheets and captures workflow errors into an error log sheet.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:** Monitors Gmail inbox for unread emails arriving every minute to trigger the workflow.
- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**
    - Type: Gmail Trigger (Trigger node)
    - Configuration: Monitors the INBOX label for unread emails, polling every minute.
    - Key expressions:
      - Filters: labelIds set to "INBOX", readStatus set to "unread".
    - Input: None (trigger)
    - Output: Email item JSON containing full email data (subject, text, headers, message ID, thread ID).
    - Edge cases:
      - OAuth token expiration or permission errors.
      - Gmail API rate limits.
      - Emails with non-standard formats or missing fields.
    - Credential: Gmail OAuth2

---

#### 2.2 Email Classification

- **Overview:** Uses a language model-based text classifier to categorize incoming emails into five categories: support, sales, complaints, information, or other.
- **Nodes Involved:**  
  - Mail Classifier

- **Node Details:**

  - **Mail Classifier**
    - Type: Text Classifier (Langchain)
    - Configuration:
      - Input text concatenates email subject and body text.
      - Categories defined with descriptions for support, sales, complaints, information, and fallback "other".
    - Key expressions:
      - Input text: `"Subject: {{subject}}\nText: {{text}}"` from Gmail Trigger node.
      - Categories configured with clear definitions.
    - Input: Output from Gmail Trigger node.
    - Output: Classification result with predicted category.
    - Edge cases:
      - Classification fallback to "other" if confidence is low.
      - Possible API errors or timeouts.
    - Credential: None (uses internal Langchain config)

---

#### 2.3 Decision Setting & Labeling

- **Overview:** Sets an internal "Decision" variable based on classifier output and applies corresponding Gmail labels for categorization.
- **Nodes Involved:**  
  - compliants label  
  - info label  
  - other label  
  - Set 2 (Decision = "Compliants")  
  - Set 3 (Decision = "Info")  
  - Set 4 (Decision = "Other")  
  - Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note7, Sticky Note8, Sticky Note9 (documentation)

- **Node Details:**

  - **compliants label**
    - Type: Gmail API Node
    - Operation: Adds label "Complaints" to the email by messageId.
    - Label ID: Specific to complaints label.
    - Input: Message ID from Gmail Trigger.
    - Output: Passes to Set 2.
    - Credential: Gmail OAuth2

  - **info label**
    - Type: Gmail API Node
    - Operation: Adds label "Information" to the email.
    - Label ID: Specific to information label.
    - Input: Message ID from Gmail Trigger.
    - Output: Passes to Set 3.
    - Credential: Gmail OAuth2

  - **other label**
    - Type: Gmail API Node
    - Operation: Adds label "Unread" (or other) to the email.
    - Label ID: UNREAD label.
    - Input: Message ID from Gmail Trigger.
    - Output: Passes to Set 4.
    - Credential: Gmail OAuth2

  - **Set 2 / Set 3 / Set 4**
    - Type: Set node
    - Configuration: Assigns string value to "Decision" field:
      - Set 2: Decision = "Compliants"
      - Set 3: Decision = "Info"
      - Set 4: Decision = "Other"
    - Input: From respective label nodes.
    - Output: Passes to No Operation nodes.

  - **Sticky Notes**
    - Serve as documentation labels for clarity on labeling and decision setting blocks.

- **Edge Cases:**
  - Label IDs must exist in Gmail account; misconfigured labels cause failures.
  - Message ID missing or invalid.
  - Gmail API rate or authorization errors.

---

#### 2.4 AI Response Generation

- **Overview:** For emails classified as Support or Sales, generates a professional draft email reply using OpenAI GPT-4.1-mini model with prompt instructions.
- **Nodes Involved:**  
  - Message a model: Support  
  - Message a model: Sales  
  - Set (for Support)  
  - Set 1 (for Sales)  
  - Sticky Note10, Sticky Note11, Sticky Note12, Sticky Note13 (documentation)

- **Node Details:**

  - **Message a model: Support**
    - Type: OpenAI Chat Completion Node
    - Configuration:
      - Model: GPT-4.1-mini
      - Messages: System and user prompt to generate professional support reply.
      - Output expected in valid JSON with Subject and Body.
      - Personalized using sender’s name extracted from email headers.
    - Input: Classified email from Mail Classifier (Support branch).
    - Output: JSON with email subject and body for Support.
    - Credential: OpenAI API (OpenRouter Account)
    - Edge cases:
      - OpenAI API quota limits or failures.
      - Prompt formatting errors.
      - Invalid JSON response.

  - **Message a model: Sales**
    - Type: OpenAI Chat Completion Node
    - Configuration:
      - Same as Support but tailored prompt for sales inquiry reply.
      - Output JSON with Subject and Body.
    - Input: Classified email from Mail Classifier (Sales branch).
    - Output: JSON with email subject and body for Sales.
    - Credential: OpenAI API (OpenRouter Account)
    - Edge cases: same as Support.

  - **Set** (Support)
    - Type: Set Node
    - Configuration: Assigns variables from AI output to "Decision" = "Sales" (likely a mislabel, should be "Support"), "Subject_email", "Body_email".
    - Input: From Create a draft: Support (see next block).
    - Output: No Operation node.

  - **Set 1** (Sales)
    - Type: Set Node
    - Configuration: Assigns variables from AI output to "Decision" = "Sales", "Subject_email", "Body_email".
    - Input: From Create a draft: Sales (see next block).
    - Output: No Operation node.

  - **Sticky Notes**: Document the automatic response generation blocks.

---

#### 2.5 Draft Creation and Thread Labeling

- **Overview:** Creates Gmail draft replies for Support and Sales emails and applies appropriate thread labels.
- **Nodes Involved:**  
  - Create a draft: Support  
  - Create a draft: Sales  
  - Add label to thread  
  - Add label to thread1

- **Node Details:**

  - **Create a draft: Support**
    - Type: Gmail Node (Create draft)
    - Configuration:
      - Subject and message body from AI output.
      - Sends draft to original sender.
      - Uses threadId to keep email in context.
    - Input: Output from Message a model: Support.
    - Output: Passes to Set node.
    - Credential: Gmail OAuth2

  - **Create a draft: Sales**
    - Type: Gmail Node (Create draft)
    - Configuration:
      - Similar to Support but for Sales replies.
    - Input: Output from Message a model: Sales.
    - Output: Passes to Set 1 node.
    - Credential: Gmail OAuth2

  - **Add label to thread**
    - Type: Gmail Node
    - Operation: Adds a specific label to the email thread.
    - Used in Sales branch.
    - Input: Gmail threadId from Trigger.
    - Credential: Gmail OAuth2

  - **Add label to thread1**
    - Type: Gmail Node
    - Operation: Adds a specific label to the email thread.
    - Used in Support branch.
    - Input: Gmail threadId.
    - Credential: Gmail OAuth2

- **Edge cases:**
  - Draft creation may fail due to API limits or malformed data.
  - ThreadId missing or invalid.
  - Label IDs must exist.

---

#### 2.6 Logging & Error Handling

- **Overview:** Logs email processing results and errors into Google Sheets for audit and monitoring.
- **Nodes Involved:**  
  - Google Sheets  
  - Google Sheets2  
  - Error Trigger  
  - Append row in sheet  
  - No Operation, do nothing  
  - No Operation, do nothing1  
  - Sticky Note, Sticky Note1, Sticky Note6 (documentation)

- **Node Details:**

  - **Google Sheets / Google Sheets2**
    - Type: Google Sheets Append Row
    - Configuration:
      - Append email data with columns: Decision, Original Email, Output Email (for Google Sheets)
      - Append email data with columns: Decision, Original Email (for Google Sheets2)
      - Both write to the same spreadsheet documentId but different sheets.
    - Input: From No Operation nodes connected to Set nodes after draft/labeling.
    - Credential: Google Sheets OAuth2
    - Edge cases:
      - Sheet IDs or document IDs invalid.
      - Google Sheets API quota or permission errors.

  - **Error Trigger**
    - Type: Error Trigger (special trigger node)
    - Configuration: Captures any workflow execution errors.
    - Output: Passes error details to Append row in sheet.

  - **Append row in sheet**
    - Type: Google Sheets Append Row
    - Configuration:
      - Appends error details: Node with Error, Error Message, Execution ID, Workflow ID, Timestamp.
      - Writes to "Errors" sheet.
    - Credential: Google Sheets OAuth2

  - **No Operation Nodes**
    - Used as connectors to structure flow and separate operations.

  - **Sticky Notes**
    - Provide documentation for this block.

- **Edge cases:**
  - Error Trigger only fires on workflow errors.
  - Sheets API permission or connectivity issues.
  - Data format mismatches.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                              | Input Node(s)              | Output Node(s)                                | Sticky Note                                                                                                                        |
|-------------------------|----------------------------|----------------------------------------------|----------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger           | Gmail Trigger (Trigger)     | Detect new unread Gmail emails                 | None                       | Mail Classifier                              |                                                                                                                                   |
| Mail Classifier         | Text Classifier (Langchain) | Classify email into categories                 | Gmail Trigger              | Message a model: Support, Message a model: Sales, compliants label, info label, other label |                                                                                                                                   |
| compliants label        | Gmail                       | Add "Complaints" label to email                | Mail Classifier            | Set 2                                        | ## Label Mail                                                                                                                     |
| info label              | Gmail                       | Add "Information" label to email               | Mail Classifier            | Set 3                                        | ## Label Mail                                                                                                                     |
| other label             | Gmail                       | Add "Unread"/Other label to email              | Mail Classifier            | Set 4                                        | ## Label Mail                                                                                                                     |
| Set 2                   | Set                         | Set Decision = "Compliants"                     | compliants label           | No Operation, do nothing                      | ## Set the decision                                                                                                               |
| Set 3                   | Set                         | Set Decision = "Info"                           | info label                 | No Operation, do nothing                      | ## Set the decision                                                                                                               |
| Set 4                   | Set                         | Set Decision = "Other"                          | other label                | No Operation, do nothing                      | ## Set the decision                                                                                                               |
| Message a model: Support | OpenAI (Langchain)          | Generate support reply draft content            | Mail Classifier            | Add label to thread1, Create a draft: Support | ## Automatic Response Support                                                                                                    |
| Message a model: Sales   | OpenAI (Langchain)          | Generate sales reply draft content              | Mail Classifier            | Create a draft: Sales, Add label to thread   | ## Automatic Response Sales                                                                                                      |
| Add label to thread1    | Gmail                       | Add support label to email thread               | Message a model: Support   | Create a draft: Support                       |                                                                                                                                   |
| Add label to thread     | Gmail                       | Add sales label to email thread                 | Message a model: Sales     | No further connection                         |                                                                                                                                   |
| Create a draft: Support | Gmail                       | Create draft email for support reply            | Add label to thread1       | Set                                          |                                                                                                                                   |
| Create a draft: Sales   | Gmail                       | Create draft email for sales reply              | Message a model: Sales     | Set 1                                         |                                                                                                                                   |
| Set                     | Set                         | Assign Support reply details for logging        | Create a draft: Support    | No Operation, do nothing1                      | ## Set the decision                                                                                                               |
| Set 1                   | Set                         | Assign Sales reply details for logging           | Create a draft: Sales      | No Operation, do nothing1                      | ## Set the decision                                                                                                               |
| No Operation, do nothing| NoOp                        | Pass data to Google Sheets2 for logging          | Set 2, Set 3, Set 4        | Google Sheets2                                |                                                                                                                                   |
| No Operation, do nothing1| NoOp                       | Pass data to Google Sheets for logging           | Set, Set 1                 | Google Sheets                                 |                                                                                                                                   |
| Google Sheets2          | Google Sheets               | Append email logs: Decision and Original Email  | No Operation, do nothing   | None                                          |                                                                                                                                   |
| Google Sheets           | Google Sheets               | Append email logs: Decision, Original, Output Email | No Operation, do nothing1  | None                                          |                                                                                                                                   |
| Error Trigger           | Error Trigger               | Catch workflow errors                            | None                       | Append row in sheet                           |                                                                                                                                   |
| Append row in sheet     | Google Sheets               | Append error details to Errors sheet             | Error Trigger              | None                                          |                                                                                                                                   |
| Sticky Note(s)          | Sticky Note                 | Provide descriptive documentation                | N/A                        | N/A                                           | Covers blocks: Label Mail, Set the decision, Automatic Response Support & Sales, General workflow overview and instructions      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**
   - Type: Gmail Trigger
   - Configure to monitor label "INBOX" for unread emails.
   - Poll every minute.
   - Connect Gmail OAuth2 credentials.

2. **Create Mail Classifier Node:**
   - Type: Text Classifier (Langchain)
   - Input text: combine incoming email subject and text.
   - Define categories: support, sales, complaints, information, other (fallback).
   - Connect output of Gmail Trigger to this node.

3. **Create Labeling Gmail Nodes:**
   - For Complaints:
     - Gmail node, operation "addLabels" on messageId.
     - Use Complaints labelId.
     - Input: from Mail Classifier node’s Complaints output.
   - For Information:
     - Gmail node, same as above, use Information labelId.
   - For Other:
     - Gmail node, add UNREAD or other generic labelId.

4. **Create Set Nodes for Decisions:**
   - For Complaints: Set Decision = "Compliants".
   - For Info: Set Decision = "Info".
   - For Other: Set Decision = "Other".
   - Connect outputs of labeling nodes to respective Set nodes.

5. **Create OpenAI Nodes for AI Response:**
   - Support branch:
     - OpenAI Chat Completion node.
     - Use GPT-4.1-mini model.
     - Prompt: instruct to generate professional support reply in JSON with Subject and Body.
     - Input: from Mail Classifier’s Support output.
   - Sales branch:
     - OpenAI Chat Completion node.
     - Similar setup with prompt tailored for sales reply.
     - Input: Mail Classifier’s Sales output.

6. **Create Gmail Draft Nodes:**
   - For Support:
     - Create draft email with subject and body from AI output.
     - SendTo original sender’s email.
     - Use threadId from Gmail Trigger.
   - For Sales:
     - Same as above.

7. **Create Gmail Label Thread Nodes:**
   - Add appropriate labels to email threads to reflect Support or Sales processing.

8. **Create Set Nodes to Assign Logging Variables:**
   - Extract AI reply Subject and Body into variables.
   - Assign "Decision" string ("Support" or "Sales").

9. **Create No Operation Nodes:**
   - Use No Operation nodes as pass-throughs for clean flow.

10. **Create Google Sheets Nodes for Logging:**
    - Append row with original email and decision to the first sheet.
    - Append row with decision, original email, and output email (reply) to second sheet.
    - Connect No Operation nodes to these Google Sheets append operations.
    - Configure Google Sheets OAuth2 credentials.
    - Specify documentId and sheetNames (Logs and Errors sheets).

11. **Create Error Handling:**
    - Add Error Trigger node.
    - Connect to Google Sheets Append Node configured to log errors with columns (Node with Error, Error Message, Execution ID, Workflow ID, Time).
    - Use same Google Sheets OAuth2 credentials with error sheetId.

12. **Add Sticky Notes:**
    - Place descriptive Sticky Notes at key logical blocks for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates classification and draft response generation for Gmail emails using OpenAI GPT models and Google Sheets logging. Includes error handling and detailed logging for audit purposes.                                                                                                                                                            | General workflow purpose.                                                                                                                                                                                                                     |
| Requires configuration of Gmail OAuth2 credentials, OpenAI API credentials (OpenRouter recommended), and Google Sheets OAuth2 with appropriate spreadsheet and sheets created for logs and errors.                                                                                                                                                             | Credential setup instructions.                                                                                                                                                                                                                |
| Label IDs in Gmail must be pre-created and correctly referenced in labeling nodes.                                                                                                                                                                                                                                                                           | Gmail Label management.                                                                                                                                                                                                                        |
| Prompts for OpenAI nodes are designed to output strictly valid JSON with Subject and Body fields to enable automated draft creation.                                                                                                                                                                                                                         | OpenAI prompt design.                                                                                                                                                                                                                          |
| Error Trigger node must be enabled to capture workflow execution errors and log them in Google Sheets for monitoring.                                                                                                                                                                                                                                        | Error handling configuration.                                                                                                                                                                                                                  |
| [OpenAI API Documentation](https://platform.openai.com/docs/) and [Gmail API Reference](https://developers.google.com/gmail/api) provide further details on API usage.                                                                                                                                                                                          | API references for advanced customization.                                                                                                                                                                                                    |
| [Google Sheets API Documentation](https://developers.google.com/sheets/api) useful for customizing sheet operations and understanding quotas.                                                                                                                                                                                                               | Google Sheets API documentation.                                                                                                                                                                                                               |
| Sticky Notes throughout the workflow provide helpful inline documentation and best practice reminders.                                                                                                                                                                                                                                                       | Workflow node comments.                                                                                                                                                                                                                        |

---

**Disclaimer:** This text is exclusively generated from an automated n8n workflow. It respects all content policies and contains no illegal or offensive elements. All data processed is legal and publicly accessible.