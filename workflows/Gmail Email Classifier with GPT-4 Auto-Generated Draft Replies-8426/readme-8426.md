Gmail Email Classifier with GPT-4 Auto-Generated Draft Replies

https://n8nworkflows.xyz/workflows/gmail-email-classifier-with-gpt-4-auto-generated-draft-replies-8426


# Gmail Email Classifier with GPT-4 Auto-Generated Draft Replies

### 1. Workflow Overview

This workflow automates the classification and response drafting of incoming Gmail emails using GPT-4. It is designed for professional environments where incoming emails need to be triaged into categories such as High Priority, Inquiry, and Finance/Billing. For each category, it automatically generates a draft reply leveraging OpenAI’s GPT-4 models, then saves those drafts in Gmail for review or sending.

**Target Use Cases:**  
- Customer support teams handling inquiries  
- Executives or assistants managing urgent emails  
- Finance teams processing billing-related communications  

**Logical Blocks:**  
- **1.1 Input Reception:** Trigger on new Gmail emails every 15 minutes.  
- **1.2 Email Classification:** Use a text classifier to determine email category.  
- **1.3 Category Labeling:** Apply Gmail labels based on classification.  
- **1.4 Draft Generation:** Generate AI-powered draft replies for each category.  
- **1.5 Draft Saving:** Save the generated drafts back to Gmail threads.  
- **1.6 Support Model Node:** A separate LLM node for support or fallback classification (not connected in main flow).  
- **1.7 Configuration Notes:** Sticky notes providing guidance on credentials and label IDs.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow when new emails arrive in Gmail, polling every 15 minutes.

**Nodes Involved:**  
- Gmail Trigger  
- Sticky Note (Credentials reminder)

**Node Details:**  

- **Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Watches Gmail inbox for new emails.  
  - Configuration: Polls every 15 minutes, no filters applied (fetches all new emails).  
  - Input: None (trigger node)  
  - Output: Email JSON data including `id`, `subject`, `text`, `threadId`.  
  - Edge Cases: Gmail API rate limits, connectivity/auth failures. Polling interval should be adjusted based on expected email volume and API quota.  
  - Credentials: Requires Gmail OAuth2 credentials.  
- **Sticky Note ("Connect your Gmail account under \"Credentials\".")**  
  - Purpose: Reminds user to configure Gmail OAuth2 credentials.

---

#### 1.2 Email Classification

**Overview:**  
Classifies the email text into predefined categories using a Langchain text classifier node.

**Nodes Involved:**  
- Gmail Category Classifier  
- LLM Support Model (not connected in main flow)

**Node Details:**  

- **Gmail Category Classifier**  
  - Type: Langchain Text Classifier  
  - Role: Categorizes email text into one of four categories: High Priority, Promotions, Inquiry, Finance/Billing.  
  - Configuration:  
    - Input text is the email body extracted as `{{$json.text}}`.  
    - Categories are clearly defined with descriptions to guide classification logic.  
  - Input: Output from Gmail Trigger (email text).  
  - Output: Classification result routed to different label nodes based on category.  
  - Potential Failures: Incorrect classification if input text is malformed or very short; API call failures; model version compatibility.  
  - Version: 1.1  
- **LLM Support Model**  
  - Type: Langchain OpenAI Chat Model (GPT-4.1-mini)  
  - Role: Possibly used as a fallback or support model for classification (not wired in main connections).  
  - Configuration: Model set to GPT-4.1-mini.  
  - Edge Cases: Currently unused, can be integrated for enhanced classification or fallback.

---

#### 1.3 Category Labeling

**Overview:**  
Applies Gmail labels to the incoming email based on classification outcome.

**Nodes Involved:**  
- Label: High Priority  
- Label: Inquiry  
- Label: Finance/Billing  
- Sticky Note (Label IDs reminder)

**Node Details:**  

- **Label: High Priority**  
  - Type: Gmail Node  
  - Role: Adds "High Priority" Gmail label to the email message.  
  - Configuration: Label ID placeholder `YOUR_LABEL_ID_HIGH_PRIORITY` to be replaced with actual Gmail label ID. Uses incoming message ID.  
  - Input: From Gmail Category Classifier main output index 0 (High Priority branch).  
  - Output: Connected to "Generate Draft: High Priority" node.  
  - Potential Failures: Invalid label ID, Gmail API permission issues, message ID missing.  
- **Label: Inquiry**  
  - Similar to above, label ID `YOUR_LABEL_ID_INQUIRY`.  
  - Input: From classification output index 2 (Inquiry).  
  - Output: Connected to "Generate Draft: Inquiry".  
- **Label: Finance/Billing**  
  - Label ID `YOUR_LABEL_ID_FINANCE`.  
  - Input: From classification output index 3 (Finance/Billing).  
  - Output: Connected to "Generate Draft: Finance/Billing".  
- **Sticky Note ("Replace YOUR_LABEL_ID_XXX with your Gmail label IDs (get them via Gmail → List Labels).")**  
  - Purpose: Instruction to user to input correct Gmail label IDs.

---

#### 1.4 Draft Generation

**Overview:**  
Generates draft email replies using GPT-4 tailored prompts for each category.

**Nodes Involved:**  
- Generate Draft: High Priority  
- Generate Draft: Inquiry  
- Generate Draft: Finance/Billing  
- Sticky Note (OpenAI API key reminder)

**Node Details:**  

- **Generate Draft: High Priority**  
  - Type: Langchain OpenAI (gpt-4o-mini)  
  - Role: Drafts fast, executive assistant style replies for urgent emails.  
  - Configuration:  
    - Model: GPT-4.0 optimized mini version.  
    - Prompt instructs to output JSON with subject and message.  
    - Input uses the original email text and subject.  
    - Output JSON parsed automatically.  
  - Input: From "Label: High Priority" node.  
  - Output: To "Save Draft: High Priority".  
  - Edge Cases: Model timeouts, incomplete JSON output, token limits.  
- **Generate Draft: Inquiry**  
  - Role: Draft polite customer support replies.  
  - Similar configuration with instructions for polite, complete replies including summary, answer, next steps, and signature.  
  - Input: From "Label: Inquiry".  
  - Output: To "Save Draft: Inquiry".  
- **Generate Draft: Finance/Billing**  
  - Role: Draft precise finance-related replies confirming key details and requesting missing docs.  
  - Input: From "Label: Finance/Billing".  
  - Output: To "Save Draft: Finance/Billing".  
- **Sticky Note ("Add your OpenAI API Key under \"Credentials\".")**  
  - Reminder to configure OpenAI API credentials for the Langchain nodes.

---

#### 1.5 Draft Saving

**Overview:**  
Saves the generated draft replies into Gmail threads as drafts, preserving the conversation context.

**Nodes Involved:**  
- Save Draft: High Priority  
- Save Draft: Inquiry  
- Save Draft: Finance/Billing

**Node Details:**  

- **Save Draft: High Priority**  
  - Type: Gmail Node (Draft)  
  - Role: Saves the generated draft message and subject to Gmail drafts under the same email thread.  
  - Configuration:  
    - Message body and subject extracted from the JSON output of the draft generation node.  
    - Thread ID set to original email's thread ID to maintain context.  
  - Input: From "Generate Draft: High Priority".  
  - Output: Workflow ends here for this branch.  
  - Edge Cases: Gmail API failures, invalid thread ID, malformed message content.  
- **Save Draft: Inquiry**  
  - Same as above but for Inquiry drafts.  
- **Save Draft: Finance/Billing**  
  - Same as above but for Finance/Billing drafts.

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                         | Input Node(s)            | Output Node(s)                   | Sticky Note                                                                                   |
|---------------------------|------------------------------|---------------------------------------|--------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Gmail Trigger             | Gmail Trigger                | Trigger on new incoming Gmail emails  | —                        | Gmail Category Classifier        | Connect your Gmail account under "Credentials".                                              |
| Gmail Category Classifier | Langchain Text Classifier    | Classify email into categories         | Gmail Trigger            | Label: High Priority, Label: Inquiry, Label: Finance/Billing |                                                                                              |
| Label: High Priority      | Gmail                       | Add High Priority Gmail label          | Gmail Category Classifier | Generate Draft: High Priority    | Replace YOUR_LABEL_ID_XXX with your Gmail label IDs (get them via Gmail → List Labels).      |
| Label: Inquiry            | Gmail                       | Add Inquiry Gmail label                 | Gmail Category Classifier | Generate Draft: Inquiry          | Replace YOUR_LABEL_ID_XXX with your Gmail label IDs (get them via Gmail → List Labels).      |
| Label: Finance/Billing    | Gmail                       | Add Finance/Billing Gmail label         | Gmail Category Classifier | Generate Draft: Finance/Billing  | Replace YOUR_LABEL_ID_XXX with your Gmail label IDs (get them via Gmail → List Labels).      |
| Generate Draft: High Priority | Langchain OpenAI (GPT-4o-mini) | Generate High Priority draft reply     | Label: High Priority     | Save Draft: High Priority        | Add your OpenAI API Key under "Credentials".                                                 |
| Generate Draft: Inquiry   | Langchain OpenAI (GPT-4o-mini) | Generate Inquiry draft reply            | Label: Inquiry           | Save Draft: Inquiry              | Add your OpenAI API Key under "Credentials".                                                 |
| Generate Draft: Finance/Billing | Langchain OpenAI (GPT-4o-mini) | Generate Finance/Billing draft reply   | Label: Finance/Billing   | Save Draft: Finance/Billing      | Add your OpenAI API Key under "Credentials".                                                 |
| Save Draft: High Priority | Gmail                       | Save generated draft to Gmail           | Generate Draft: High Priority | —                              |                                                                                              |
| Save Draft: Inquiry       | Gmail                       | Save generated draft to Gmail           | Generate Draft: Inquiry  | —                               |                                                                                              |
| Save Draft: Finance/Billing | Gmail                     | Save generated draft to Gmail           | Generate Draft: Finance/Billing | —                            |                                                                                              |
| LLM Support Model         | Langchain OpenAI Chat       | Support/fallback LLM model (unused)     | —                        | Gmail Category Classifier (ai_languageModel input) |                                                                                              |
| Sticky Note1              | Sticky Note                 | Reminder about OpenAI API Key           | —                        | —                               | Add your OpenAI API Key under "Credentials".                                                 |
| Sticky Note2              | Sticky Note                 | Reminder about Gmail label IDs          | —                        | —                               | Replace YOUR_LABEL_ID_XXX with your Gmail label IDs (get them via Gmail → List Labels).      |
| Sticky Note               | Sticky Note                 | Reminder about Gmail credentials        | —                        | —                               | Connect your Gmail account under "Credentials".                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set to poll every 15 minutes (mode: everyX, unit: minutes, value: 15)  
   - No filters (to capture all incoming emails)  
   - Connect Gmail OAuth2 credentials.

2. **Create Gmail Category Classifier Node**  
   - Type: Langchain Text Classifier  
   - Set input text to: `{{$json.text}}` from Gmail Trigger output.  
   - Define categories:  
     - High Priority (urgent matters, escalations)  
     - Promotions (newsletters, ads)  
     - Inquiry (questions/requests)  
     - Finance/Billing (invoices, payments)  
   - Connect output of Gmail Trigger to this node's input.

3. **Create Gmail Label Nodes for Each Category**  
   - For each category (High Priority, Inquiry, Finance/Billing), create a Gmail node:  
     - Operation: Add Labels  
     - Label IDs: Replace placeholders with your actual Gmail label IDs.  
     - Message ID: `={{ $json.id }}` from classifier output.  
   - Connect classifier outputs accordingly:  
     - High Priority → Label: High Priority  
     - Inquiry → Label: Inquiry  
     - Finance/Billing → Label: Finance/Billing

4. **Create Generate Draft Nodes for Each Category**  
   - Type: Langchain OpenAI (choose GPT-4o-mini model)  
   - Configure the prompt messages specifically for each category, including instructions to output JSON with `subject` and `message`.  
   - Example prompt content:  
     - High Priority: Executive assistant style, short summary, clarifying questions, provisional answers.  
     - Inquiry: Polite customer support reply with summary, direct answer, next steps, FAQ links, signature.  
     - Finance/Billing: Confirm key financial fields, request missing documents, state due dates.  
   - Connect each Label node to its corresponding Generate Draft node.

5. **Create Save Draft Nodes for Each Category**  
   - Type: Gmail  
   - Operation: Save draft (resource: draft)  
   - Configure:  
     - Message: `{{ JSON.parse($json.message.content)['message'] || $json.message.content }}`  
     - Subject: `{{ JSON.parse($json.message.content)['subject'] || \`Re: ${$('Gmail Trigger').item.json.subject}\` }}`  
     - Options: Thread ID set to `={{ $('Gmail Trigger').item.json.threadId }}`  
   - Connect each Generate Draft node to its respective Save Draft node.

6. **Credentials Setup**  
   - Add Gmail OAuth2 credentials under n8n credentials.  
   - Add OpenAI API key credentials for Langchain nodes.

7. **Optional: Add LLM Support Model Node**  
   - Create Langchain OpenAI Chat node (GPT-4.1-mini) for supporting classification or fallback.  
   - Currently disconnected; can be integrated for enhanced AI tasks.

8. **Add Sticky Notes**  
   - Remind to configure Gmail credentials.  
   - Remind to add OpenAI API key.  
   - Remind to replace Gmail label ID placeholders.

9. **Connect all nodes as per the flow:**  
   - Gmail Trigger → Gmail Category Classifier  
   - Gmail Category Classifier → Label nodes per category  
   - Label nodes → Generate Draft nodes  
   - Generate Draft nodes → Save Draft nodes

---

### 5. General Notes & Resources

| Note Content                                                                            | Context or Link                                                        |
|-----------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Replace placeholder Gmail label IDs with actual IDs retrieved via Gmail's List Labels API or Gmail UI. | Label nodes require valid Gmail label IDs for correct labeling.      |
| Add your OpenAI API key under the "Credentials" section in n8n to enable GPT-4 usage.    | Required for Langchain OpenAI nodes to function.                      |
| Configure Gmail OAuth2 credentials properly to allow read/write operations on emails.   | Essential for Gmail Trigger, Labeling, and Draft saving functionality. |
| Workflow polls Gmail every 15 minutes; adjust frequency based on email volume and API quotas. | Avoid hitting Gmail API rate limits.                                  |
| Drafts are saved, not sent automatically, allowing manual review before sending replies. | Enables quality control and editing.                                  |
| GPT-4o-mini and GPT-4.1-mini models are optimized variants; verify availability in your OpenAI plan. | Model availability and pricing may vary.                             |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.