Filter fraudulent leads with GPT-4o-mini, AbstractAPI, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/filter-fraudulent-leads-with-gpt-4o-mini--abstractapi--google-sheets-and-slack-13774


# Filter fraudulent leads with GPT-4o-mini, AbstractAPI, Google Sheets and Slack

# Workflow Reference: AI Fraud Detection & Budget Guard

This document details an n8n workflow designed to protect lead generation funnels by filtering out fraudulent submissions. It uses a multi-layered approach—combining IP intelligence with AI content analysis—to verify leads before they reach sales teams or CRM systems.

---

### 1. Workflow Overview

The workflow acts as a security gate for incoming webhooks (e.g., from Tally.so or other form builders). It simultaneously analyzes the technical origin of the request (IP address) and the semantic quality of the message (AI analysis). By identifying VPNs, proxies, and bot-generated text, it prevents "garbage" data from polluting sales pipelines and helps optimize marketing spend by creating exclusion audiences.

**Logical Blocks:**
- **1.1 Input Reception:** Captures lead data via a webhook.
- **1.2 Parallel Verification:** Performs an IP security check (AbstractAPI) and an AI intent analysis (OpenAI).
- **1.3 Data Consolidation:** Merges results from both checks.
- **1.4 Decision Logic:** Evaluates results to determine if the lead is "Spam" or "Human."
- **1.5 Action & Logging:** Routes leads to specific Google Sheets and sends alerts via Slack for neutralized threats.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Receives the lead submission data from an external form provider.
- **Nodes Involved:** `Webhook`
- **Node Details:**
    - **Node Name:** Webhook
    - **Type:** Webhook Node
    - **Configuration:** Set to `POST` method with path `tally-leads`.
    - **Input/Output:** Receives raw JSON from the form provider; outputs fields like Name, Email, Message, and the IP address (usually found in headers like `cf-connecting-ip`).
    - **Edge Cases:** Missing headers or malformed JSON payloads from the source.

#### 2.2 Parallel Verification
- **Overview:** Executes two independent checks to verify the lead's legitimacy.
- **Nodes Involved:** `HTTP Request`, `Message a model` (OpenAI)
- **Node Details:**
    - **Node Name:** HTTP Request
        - **Role:** IP Intelligence check via AbstractAPI.
        - **Config:** Queries `https://ip-intelligence.abstractapi.com/v1`.
        - **Expressions:** Uses `api_key` and a hardcoded or dynamic `ip_address`.
        - **Failure Type:** API rate limits or invalid API key.
    - **Node Name:** Message a model
        - **Role:** Analyzes the name and message for bot-like patterns.
        - **Model:** `gpt-4o-mini`.
        - **System Prompt:** Instructed to return only "Spam" or "Human".
        - **Input Expression:** `Name: {{ $json.body.data.fields[0].value }} | Message: {{ $json.body.data.fields[2].value }}`.

#### 2.3 Data Consolidation & Decision
- **Overview:** Combines the security data and AI verdict into one object to apply filtering logic.
- **Nodes Involved:** `Merge`, `If`
- **Node Details:**
    - **Node Name:** Merge
        - **Role:** Combines the output of the IP check and the AI check by position.
    - **Node Name:** If
        - **Logic:** Uses **OR** logic.
        - **Conditions:** 
            1. `is_vpn` is `true` 
            2. `is_proxy` is `true` 
            3. AI Verdict equals `spam`.
        - **Output:** `true` (Spam detected) or `false` (Verified lead).

#### 2.4 Action & Logging
- **Overview:** Archives the lead in the appropriate spreadsheet and notifies the team if spam is blocked.
- **Nodes Involved:** `Potential Spam` (Google Sheets), `Verified Leads` (Google Sheets), `Spam Alert` (Slack)
- **Node Details:**
    - **Node Name:** Potential Spam / Verified Leads
        - **Type:** Google Sheets (Append operation).
        - **Fields:** Maps Name, Email, IP, AI Verdict, and Security Flags.
    - **Node Name:** Spam Alert
        - **Type:** Slack (Send Message).
        - **Configuration:** Sends a notification to a specific user/channel when a lead is neutralized.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | Webhook | Trigger / Input | None | HTTP Request, Message a model | Trigger: The workflow begins with a Webhook node... |
| HTTP Request | HTTP Request | IP Security Check | Webhook | Merge | This node queries AbstractAPI using the lead's IP address... |
| Message a model | OpenAI | AI Content Analysis | Webhook | Merge | Simultaneously, the lead's name and message are sent to GPT-4o-mini... |
| Merge | Merge | Data Joiner | HTTP Request, Message a model | If | This node waits for both the technical check and the AI analysis to finish. |
| If | If | Fraud Filter Logic | Merge | Potential Spam, Verified Leads | This is the core decision engine. It uses "OR" logic... |
| Potential Spam | Google Sheets | Fraud Logging | If (True) | Spam Alert | Flagged leads are appended to the "Potential Spam" Google Sheet. |
| Verified Leads | Google Sheets | Lead Management | If (False) | None | Legitimate leads are saved to the "Verified leads" sheet. |
| Spam Alert | Slack | Team Notification | Potential Spam | None | A notification is sent to Slack to inform the team... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Webhook** node. Set the path to `tally-leads` and HTTP Method to `POST`.
2.  **IP Intelligence:** Add an **HTTP Request** node.
    - URL: `https://ip-intelligence.abstractapi.com/v1`
    - Method: `GET`
    - Query Parameters: `api_key` (your AbstractAPI key) and `ip_address` (referenced from the Webhook header/body).
3.  **AI Analysis:** Add an **OpenAI** node (Message a model).
    - Model: `gpt-4o-mini`.
    - System Message: Command the AI to output ONLY "Spam" or "Human".
    - Prompt: Pass the Name and Message fields from the Webhook node.
4.  **Consolidation:** Connect both the HTTP Request and OpenAI nodes to a **Merge** node (Mode: Combine by Position).
5.  **Filtering Logic:** Connect the Merge node to an **If** node.
    - Condition 1: String `is_vpn` equals `true`.
    - Condition 2: String `is_proxy` equals `true`.
    - Condition 3: String (AI Output) equals `spam` (ignore case).
    - Set Combinator to **OR**.
6.  **Storage:** 
    - On the **True** branch, add a **Google Sheets** node (Append) pointing to a "Potential Spam" tab.
    - On the **False** branch, add a **Google Sheets** node (Append) pointing to a "Verified Leads" tab.
7.  **Notification:** Connect a **Slack** node to the "Potential Spam" Google Sheets node to send a "Spam Neutralized" message.
8.  **Credentials:** Configure OAuth2 for Google Sheets and Slack; Provide API Keys for OpenAI and AbstractAPI.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Connect with the creator for support. | [LinkedIn - Nithish](https://www.linkedin.com/in/bynithish/) |
| Google Sheets Schema Requirements: Tab 1 must include Name, Email, IP, AI Verdict, Security Flags, Timestamp. | Configuration Requirement |
| Google Sheets Schema Requirements: Tab 2 must include Name, Email, Message, Location, Timestamp. | Configuration Requirement |
| Use "Potential Spam" sheet as an exclusion audience for Meta/Google Ads. | Marketing Strategy |