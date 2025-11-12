Automatically Classify and Label Gmail Emails with Google Gemini AI

https://n8nworkflows.xyz/workflows/automatically-classify-and-label-gmail-emails-with-google-gemini-ai-3772


# Automatically Classify and Label Gmail Emails with Google Gemini AI

### 1. Workflow Overview

This workflow automates the classification and labeling of incoming Gmail emails using AI, specifically Google Gemini or any large language model (LLM). It targets users who want to keep their inbox organized by automatically sorting emails into categories such as High Priority, Work Related, or Promotions, and applying corresponding Gmail labels without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new incoming emails from Gmail in real-time.
- **1.2 AI Classification:** Uses an AI text classifier enhanced by Google Gemini to categorize email content.
- **1.3 Conditional Labeling:** Routes classified emails to appropriate Gmail label nodes based on category.
- **1.4 Gmail Label Application:** Applies the selected Gmail label to the email.
- **1.5 Optional/Disabled AI Agents:** Additional AI nodes for advanced email handling or replies (disabled in this workflow).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow whenever a new email arrives in the connected Gmail inbox.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**

- **Gmail Trigger**  
  - *Type:* Trigger node for Gmail  
  - *Role:* Detects new incoming emails every minute.  
  - *Configuration:*  
    - Polling mode set to check every minute.  
    - No filters applied, so it triggers on all new emails.  
  - *Credentials:* Connected via OAuth2 to the user’s Gmail account.  
  - *Input:* None (trigger node).  
  - *Output:* Emits the full email data including headers, body (text and HTML), and metadata.  
  - *Edge Cases:*  
    - OAuth token expiration or revocation may cause trigger failure.  
    - Gmail API rate limits could delay polling.  
  - *Sticky Notes:*  
    - "### 1) Change to your gmail's credential" — reminds user to configure Gmail OAuth2 credentials properly.

---

#### 2.2 AI Classification

**Overview:**  
This block classifies the incoming email content into predefined categories using a text classification agent powered by Google Gemini.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Classification Agent

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type:* AI language model node (Google Gemini)  
  - *Role:* Provides AI-powered understanding and processing to support classification.  
  - *Configuration:*  
    - Model set to "models/gemini-2.0-flash-001".  
    - No additional options configured.  
  - *Credentials:* Uses Google Palm API credentials linked to a Google account.  
  - *Input:* Receives prompts from the Classification Agent node as part of the AI chain.  
  - *Output:* Returns AI-generated classification data.  
  - *Edge Cases:*  
    - API quota limits or authentication errors.  
    - Model response delays or timeouts.  
  - *Sticky Notes:*  
    - "### 2) Change to your desire LLMs" — user can switch to other LLMs if preferred.

- **Classification Agent**  
  - *Type:* Text classifier node using LangChain framework  
  - *Role:* Classifies email text into categories: High Priority, KS Work Related, Promotion, or Other.  
  - *Configuration:*  
    - System prompt instructs the agent to output classification strictly in JSON format without explanation.  
    - Input text is dynamically set to the email’s plain text or HTML content.  
    - Categories are predefined with descriptions and keywords to guide classification.  
  - *Input:* Receives email content from Gmail Trigger.  
  - *Output:* Emits classification results used for conditional routing.  
  - *Edge Cases:*  
    - Classification ambiguity if email content is unclear or overlaps categories.  
    - Expression evaluation errors if input text is missing or malformed.  
  - *Sticky Notes:*  
    - "### 4) Agent instruction: Input the category name that you just created in gmail. Description = Tell agent about how should it classify your email. Keywords can be useful to let your agent classify the email context."

---

#### 2.3 Conditional Labeling

**Overview:**  
Based on the classification result, this block routes emails to the corresponding Gmail label nodes to apply the correct label.

**Nodes Involved:**  
- Add Label (High Priority)  
- Add Label (KS Work Related)  
- Add Label Promotion

**Node Details:**

- **Add Label (High Priority)**  
  - *Type:* Gmail node for modifying emails  
  - *Role:* Adds the "High Priority" label to the email.  
  - *Configuration:*  
    - Uses Gmail label ID corresponding to "High Priority".  
    - Message ID dynamically set from the incoming email.  
    - Operation set to "addLabels".  
  - *Input:* Receives emails classified as High Priority from Classification Agent.  
  - *Output:* None (terminal node).  
  - *Edge Cases:*  
    - Label ID mismatch or deletion in Gmail causes failure.  
    - Insufficient Gmail API permissions.  

- **Add Label (KS Work Related)**  
  - *Type:* Gmail node for modifying emails  
  - *Role:* Adds the "KS Work Related" label to the email.  
  - *Configuration:*  
    - Uses Gmail label ID for KS Work Related.  
    - Message ID dynamically set.  
    - Operation "addLabels".  
  - *Input:* Receives emails classified as KS Work Related.  
  - *Output:* None.  
  - *Edge Cases:* Same as above.

- **Add Label Promotion**  
  - *Type:* Gmail node for modifying emails  
  - *Role:* Adds the "Promotion" label to the email.  
  - *Configuration:*  
    - Uses Gmail label ID for Promotion.  
    - Message ID dynamic.  
    - Operation "addLabels".  
  - *Input:* Receives emails classified as Promotion.  
  - *Output:* None.  
  - *Edge Cases:* Same as above.

- *Sticky Notes:*  
  - "### 5) Add Label Nodes: In this option 'Label Names or IDs' -> Select the category to match with the Classification Agent Node."  
  - "### 6) Add-on: You can add more category of your choice!"

---

#### 2.4 Gmail Label Application

**Overview:**  
This block is effectively integrated with the conditional labeling nodes above; each node applies the label to the email in Gmail.

**Nodes Involved:**  
- Same as Conditional Labeling nodes (Add Label nodes)

**Node Details:**  
Covered above.

---

#### 2.5 Optional/Disabled AI Agents (Not Active)

**Overview:**  
These nodes are disabled but represent potential extensions for AI-powered email replies or alternative LLM usage.

**Nodes Involved:**  
- AI Agent (LangChain agent)  
- Groq Chat Model  
- Gmail3 (Gmail draft creation)

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain AI agent node  
  - *Role:* Intended to generate professional email replies based on classified email content.  
  - *Configuration:*  
    - System message instructs the agent to write polite, professional replies as the user.  
    - Input text references classification results.  
  - *Status:* Disabled.  
  - *Edge Cases:* Requires active AI model and proper prompt tuning.

- **Groq Chat Model**  
  - *Type:* Alternative AI language model node  
  - *Role:* Could be used instead of Google Gemini for classification or reply generation.  
  - *Status:* Disabled.

- **Gmail3**  
  - *Type:* Gmail node for draft creation  
  - *Role:* Intended to create draft replies based on AI Agent output.  
  - *Status:* Disabled.

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                         | Input Node(s)          | Output Node(s)                          | Sticky Note                                                                                       |
|----------------------------|--------------------------------------|---------------------------------------|-----------------------|----------------------------------------|-------------------------------------------------------------------------------------------------|
| Gmail Trigger              | n8n-nodes-base.gmailTrigger           | Trigger on new incoming emails        | None                  | Classification Agent                   | ### 1) Change to your gmail's credential                                                        |
| Google Gemini Chat Model   | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI model for classification support   | Classification Agent (AI model input) | Classification Agent                   | ### 2) Change to your desire LLMs                                                               |
| Classification Agent       | @n8n/n8n-nodes-langchain.textClassifier | Classifies email content into categories | Gmail Trigger          | Add Label (High Priority), Add Label (KS Work Related), Add Label Promotion | ### 4) Agent instruction: Input the category name that you just created in gmail. Description = Tell agent about how should it classify your email. Keywords can be useful to let your agent classify the email context. |
| Add Label (High Priority)  | n8n-nodes-base.gmail                  | Adds High Priority label to email     | Classification Agent   | None                                   | ### 5) Add Label Nodes: In this option "Label Names or IDs" -> Select the category to match with the Classification Agent Node. |
| Add Label (KS Work Related)| n8n-nodes-base.gmail                  | Adds KS Work Related label             | Classification Agent   | None                                   | ### 5) Add Label Nodes: In this option "Label Names or IDs" -> Select the category to match with the Classification Agent Node. |
| Add Label Promotion        | n8n-nodes-base.gmail                  | Adds Promotion label                   | Classification Agent   | None                                   | ### 5) Add Label Nodes: In this option "Label Names or IDs" -> Select the category to match with the Classification Agent Node. |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent        | (Disabled) Generate reply emails      | Groq Chat Model        | Gmail3                                 |                                                                                                 |
| Groq Chat Model           | @n8n/n8n-nodes-langchain.lmChatGroq   | (Disabled) Alternative AI model       | None                  | AI Agent                              |                                                                                                 |
| Gmail3                    | n8n-nodes-base.gmail                  | (Disabled) Create draft reply emails  | AI Agent               | None                                   |                                                                                                 |
| Sticky Note               | n8n-nodes-base.stickyNote             | Instructional notes                   | None                  | None                                   | Multiple notes with setup instructions (see detailed notes in sections 2.1, 2.2, 2.3)           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Set to poll every minute (pollTimes: everyMinute).  
   - No filters (captures all new emails).  
   - Connect Gmail OAuth2 credentials for the target inbox.

2. **Create Google Gemini Chat Model Node:**  
   - Type: LangChain Google Gemini Chat Model  
   - Model: "models/gemini-2.0-flash-001"  
   - Connect Google Palm API credentials.  
   - No additional options needed.

3. **Create Classification Agent Node:**  
   - Type: LangChain Text Classifier  
   - System prompt: "Please classify the text provided by the user into one of the following categories: {categories}, and use the provided formatting instructions below. Don't explain, and only output the json."  
   - Input text: Set expression to `{{$json.text || $json.html}}` to use email content.  
   - Define categories:  
     - High Priority (keywords: urgent, ASAP, immediate, deadline, action required, high priority)  
     - KS Work Related (keywords: Kajonkietsuksa School, Kajonkietsuksa, School)  
     - Promotion (keywords: newsletter, promotion, offer, sale, campaign, marketing, launch)  
     - Other (fallback category)  
   - Connect input from Gmail Trigger node.  
   - Connect AI language model input to Google Gemini Chat Model node.

4. **Create Gmail Label Nodes:**  
   For each category, create a Gmail node configured to add the corresponding label:  
   - **Add Label (High Priority):**  
     - Operation: addLabels  
     - Label IDs: Use Gmail label ID for "High Priority" (create this label manually in Gmail if not existing).  
     - Message ID: Set expression to `{{$json.id}}` from incoming email.  
   - **Add Label (KS Work Related):**  
     - Same as above, with label ID for "KS Work Related".  
   - **Add Label Promotion:**  
     - Same as above, with label ID for "Promotion".

5. **Connect Classification Agent Outputs:**  
   - Connect Classification Agent’s main output to each Add Label node.  
   - The workflow routes classified emails to the appropriate label node based on classification results.

6. **(Optional) Add Disabled AI Agent and Reply Nodes:**  
   - Create LangChain AI Agent node with system message instructing professional reply generation.  
   - Create alternative AI model node (Groq Chat Model) if desired.  
   - Create Gmail node to draft replies.  
   - These nodes are disabled by default and can be enabled for advanced functionality.

7. **Add Sticky Notes for User Guidance:**  
   - Add notes near Gmail Trigger to remind about Gmail credential setup.  
   - Add notes near AI nodes to remind about LLM selection.  
   - Add notes near Classification Agent to explain category setup and prompt instructions.  
   - Add notes near Label nodes to explain label ID selection and possible extensions.

8. **Activate the Workflow:**  
   - Ensure all credentials are valid and connected.  
   - Activate the workflow to run continuously.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup takes less than 2 minutes and runs 24/7 to keep your inbox organized automatically.                   | Workflow description                                                                                |
| You can add more categories like "Finance" or "Newsletters" by editing the classification prompt.          | Workflow description                                                                                |
| Works best with a minimal set of categories to avoid classification overlap.                                | Workflow description                                                                                |
| Can be adapted to other LLMs such as OpenAI or Claude by swapping the AI model nodes and credentials.      | Sticky Note near Google Gemini Chat Model node                                                     |
| Gmail labels must be created manually in your Gmail inbox before running the workflow.                      | Sticky Note near Gmail Trigger and Label nodes                                                     |
| For detailed Gmail label management, refer to Gmail API documentation: https://developers.google.com/gmail/api | External resource                                                                                   |
| Google Gemini API quota and authentication must be properly configured to avoid runtime errors.             | AI nodes edge case notes                                                                            |

---

This document fully describes the workflow structure, node configurations, and setup instructions to enable both human users and automation agents to understand, reproduce, and extend the email classification and labeling automation using n8n and Google Gemini AI.