lemlist <> GPT-3: Supercharge your sales workflows

https://n8nworkflows.xyz/workflows/lemlist----gpt-3--supercharge-your-sales-workflows-1838


# lemlist <> GPT-3: Supercharge your sales workflows

### 1. Workflow Overview

This workflow integrates lemlist, OpenAI GPT-3, HubSpot CRM, and Slack to automate the classification of email responses received in lemlist campaigns and trigger corresponding actions. It targets sales and customer engagement teams who want to streamline response handling, improve lead management, and automate follow-up tasks.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Classification:** Receives email replies from lemlist via webhook, sends the email text to OpenAI GPT-3 for classification into categories (Interested, Out of Office, Unsubscribe, Other), then merges classification results with the original data.

- **1.2 Decision Routing (Switch):** Routes the classified replies into different branches depending on the category assigned by GPT-3.

- **1.3 Unsubscribe Handling:** Automatically unsubscribes leads from lemlist campaigns when the reply is an unsubscribe request.

- **1.4 Interested Lead Processing:** Marks leads as interested in lemlist, creates deals in HubSpot, and sends Slack alerts.

- **1.5 Out of Office (OOO) Follow-Up Task Creation:** Creates follow-up tasks in HubSpot for leads marked as Out of Office.

- **1.6 General Slack Notification:** Sends Slack notifications for any replies that do not fall into the previous categories ("Other").

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Classification

- **Overview:**  
  This block listens for email replies from lemlist campaigns, extracts the reply text, and uses OpenAI GPT-3 to classify the replies into predefined categories. The classification output is merged back with the original input data.

- **Nodes Involved:**  
  - Lemlist - Lead Replied  
  - OpenAI  
  - Merge

- **Node Details:**

  - **Lemlist - Lead Replied**  
    - *Type/Role:* lemlist trigger node, listens for email reply events.  
    - *Configuration:* Listens for "emailsReplied" event with `isFirst` set to true (first reply only).  
    - *Key Expressions:* None beyond standard webhook payload.  
    - *Input:* External webhook trigger from lemlist.  
    - *Output:* JSON object containing lead info and email reply text.  
    - *Edge Cases:* Missed webhook events, authentication failure with lemlist API, webhook ID mismatch.  
    - *Version:* 1  

  - **OpenAI**  
    - *Type/Role:* Uses GPT-3 to classify the reply text into categories.  
    - *Configuration:*  
      - Prompt dynamically constructed to include categories and the cleaned reply text (trimmed and line breaks removed).  
      - Parameters: topP=1, maxTokens=6, temperature=0 for deterministic classification.  
    - *Key Expressions:*  
      - `{{$json["text"].replaceAll(/^\s+|\s+$/g, '').replace(/(\r\n|\n|\r)/gm, "")}}` to sanitize input text.  
    - *Input:* Email reply text from Lemlist node.  
    - *Output:* Predicted category (e.g., "Interested", "Out of Office", "Unsubscribe", "Other").  
    - *Edge Cases:* API quota limits, network timeouts, unexpected GPT output format.  
    - *Version:* 1  

  - **Merge**  
    - *Type/Role:* Combines output from OpenAI classification with original data from Lemlist trigger.  
    - *Configuration:*  
      - Mode: Combine by position, preferring values from input 1 in case of clashes.  
    - *Input:* Receives data from OpenAI and Lemlist nodes.  
    - *Output:* Merged JSON containing original lead data plus classification result.  
    - *Edge Cases:* Data mismatch if inputs are not synchronized correctly.  
    - *Version:* 2  

#### 2.2 Decision Routing (Switch)

- **Overview:**  
  Routes the merged data into different branches based on the classification category received from GPT-3.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - *Type/Role:* Conditional logic node that directs flow based on the category string.  
    - *Configuration:*  
      - Evaluates trimmed classification text against values: "Unsubscribe", "Interested", "Out of Office".  
      - Has 4 outputs:  
        - 0: Unsubscribe  
        - 1: Interested  
        - 2: Out of Office  
        - 3: Fallback for any other category ("Other").  
    - *Key Expression:* `={{ $json["text"].trim() }}`  
    - *Input:* Merged data from Merge node.  
    - *Output:* Routes to corresponding nodes based on category.  
    - *Edge Cases:* Misclassification, unexpected category strings, case sensitivity issues.  
    - *Version:* 1  

#### 2.3 Unsubscribe Handling

- **Overview:**  
  Automatically unsubscribes leads from the lemlist campaign when they request to unsubscribe.

- **Nodes Involved:**  
  - Lemlist - Unsubscribe

- **Node Details:**

  - **Lemlist - Unsubscribe**  
    - *Type/Role:* Performs unsubscribe operation on a lead in lemlist.  
    - *Configuration:*  
      - Uses lead email and campaign ID from JSON data.  
      - Operation: "unsubscribe" on "lead" resource.  
    - *Key Expressions:*  
      - Email: `={{ $json["leadEmail"] }}`  
      - Campaign ID: `={{ $json["campaignId"] }}`  
    - *Input:* From Switch node output for "Unsubscribe" category.  
    - *Output:* Confirmation of unsubscribe action.  
    - *Edge Cases:* Invalid email, campaign ID mismatch, API authentication failure, network errors.  
    - *Version:* 1  
    - *Credentials:* lemlist API key (team API).  

#### 2.4 Interested Lead Processing

- **Overview:**  
  Marks leads as interested in lemlist, retrieves contact ID from HubSpot, creates a new deal in HubSpot, then sends a Slack alert.

- **Nodes Involved:**  
  - lemlist - Mark as interested (HTTP Request)  
  - HubSpot - Get contact ID  
  - HubSpot - Create Deal  
  - Slack  

- **Node Details:**

  - **lemlist - Mark as interested**  
    - *Type/Role:* HTTP Request node to mark lead as interested in lemlist.  
    - *Configuration:*  
      - POST request to `https://api.lemlist.com/api/campaigns/YOUR_CAMPAIGN_ID/leads/{leadEmail}/interested`.  
      - Uses lemlist API credentials.  
      - Campaign ID is hardcoded as `YOUR_CAMPAIGN_ID` — should be replaced by actual campaign ID in a production setup.  
    - *Input:* From Switch output for "Interested".  
    - *Output:* HTTP response confirming interested status.  
    - *Edge Cases:* Hardcoded campaign ID may cause failures; invalid email; API rate limits; authentication errors.  
    - *Version:* 2  
    - *Credentials:* lemlist API key.  

  - **HubSpot - Get contact ID**  
    - *Type/Role:* HubSpot API node to retrieve contact information using email.  
    - *Configuration:*  
      - Email from JSON `leadEmail`.  
      - Additional fields: `firstName` and `lastName` from JSON properties.  
    - *Input:* From Switch output for "Interested" (parallel to lemlist mark interested).  
    - *Output:* Contact details including contact VID (unique ID).  
    - *Edge Cases:* Contact not found, API limits, OAuth token expiration.  
    - *Version:* 1  
    - *Credentials:* HubSpot OAuth2.  

  - **HubSpot - Create Deal**  
    - *Type/Role:* Creates a new deal associated with the contact in HubSpot.  
    - *Configuration:*  
      - Deal stage set to "79009480" (likely a pipeline stage ID).  
      - Deal name dynamically generated using identity profile email.  
      - Associates deal with contact VID retrieved earlier.  
    - *Input:* From HubSpot - Get contact ID node.  
    - *Output:* Created deal JSON including deal ID.  
    - *Edge Cases:* Invalid stage ID, association failure, insufficient permissions.  
    - *Version:* 1  
    - *Credentials:* HubSpot OAuth2.  

  - **Slack**  
    - *Type/Role:* Sends notification message to Slack channel about new interested lead and deal.  
    - *Configuration:*  
      - Message includes dynamic link to the HubSpot deal using deal ID.  
      - Channel name must be replaced with actual Slack channel.  
    - *Input:* From HubSpot - Create Deal node.  
    - *Output:* Slack message post confirmation.  
    - *Edge Cases:* Slack API rate limits, invalid channel name, expired OAuth token.  
    - *Version:* 1  
    - *Credentials:* Slack OAuth2.  

#### 2.5 Out of Office (OOO) Follow-Up Task Creation

- **Overview:**  
  For leads classified as Out of Office, retrieves contact ID from HubSpot and creates a follow-up task associated with that contact.

- **Nodes Involved:**  
  - HubSpot - Get contact ID1  
  - follow up task

- **Node Details:**

  - **HubSpot - Get contact ID1**  
    - *Type/Role:* Same as previous HubSpot Get contact ID node, fetches contact info by email.  
    - *Configuration:* Uses lead email, first and last name from JSON.  
    - *Input:* From Switch output for "Out of Office".  
    - *Output:* Contact details.  
    - *Edge Cases:* Same as HubSpot Get contact ID node.  
    - *Version:* 1  
    - *Credentials:* HubSpot OAuth2.  

  - **follow up task**  
    - *Type/Role:* HubSpot node creating a task engagement.  
    - *Configuration:*  
      - Task subject dynamically created as: "OOO - Follow up with {firstname} {lastname}".  
      - Associated with contact via VID.  
    - *Input:* From HubSpot - Get contact ID1 node.  
    - *Output:* Task creation confirmation.  
    - *Edge Cases:* Permission issues, invalid contact IDs, API errors.  
    - *Version:* 1  
    - *Credentials:* HubSpot OAuth2.  

#### 2.6 General Slack Notification for Other Replies

- **Overview:**  
  For replies that do not match other categories (fallback), sends a Slack notification indicating a lead replied, with links to lemlist campaign reports.

- **Nodes Involved:**  
  - Slack1

- **Node Details:**

  - **Slack1**  
    - *Type/Role:* Sends Slack message for general lead reply notifications.  
    - *Configuration:*  
      - Message includes dynamic link to lemlist campaign reports using team and campaign IDs.  
      - Channel name must be replaced with actual Slack channel.  
    - *Input:* From Switch output fallback branch.  
    - *Output:* Slack message confirmation.  
    - *Edge Cases:* Invalid channel, OAuth errors, API limits.  
    - *Version:* 1  
    - *Credentials:* Slack OAuth2.  

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                               | Input Node(s)              | Output Node(s)                 | Sticky Note                                                  |
|----------------------------|-----------------------|-----------------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------|
| Lemlist - Lead Replied      | lemlistTrigger        | Receive email replies from lemlist campaigns | Webhook (external)          | OpenAI, Merge                  |                                                              |
| OpenAI                     | openAi                | Classify email reply into categories          | Lemlist - Lead Replied      | Merge                         |                                                              |
| Merge                      | merge                 | Combine original data with classification     | Lemlist - Lead Replied, OpenAI | Switch                       |                                                              |
| Switch                     | switch                | Route flow based on classification category   | Merge                      | Lemlist - Unsubscribe, lemlist - Mark as interested, HubSpot - Get contact ID1, Slack1 |                                                              |
| Lemlist - Unsubscribe       | lemlist               | Unsubscribe leads from campaigns               | Switch (Unsubscribe branch)|                               |                                                              |
| lemlist - Mark as interested| httpRequest           | Mark lead as interested in lemlist             | Switch (Interested branch) | HubSpot - Get contact ID       | Campaign ID hardcoded as YOUR_CAMPAIGN_ID, replace accordingly |
| HubSpot - Get contact ID    | hubspot               | Retrieve contact info by email                  | Switch (Interested branch) | HubSpot - Create Deal          |                                                              |
| HubSpot - Create Deal       | hubspot               | Create deal in HubSpot linked to contact       | HubSpot - Get contact ID    | Slack                        |                                                              |
| Slack                      | slack                 | Notify Slack channel about new interested lead | HubSpot - Create Deal       |                               |                                                              |
| HubSpot - Get contact ID1   | hubspot               | Retrieve contact info for OOO leads             | Switch (Out of Office branch)| follow up task               |                                                              |
| follow up task             | hubspot               | Create follow-up task for OOO leads             | HubSpot - Get contact ID1   |                               |                                                              |
| Slack1                     | slack                 | Notify Slack channel for other reply categories| Switch (Fallback branch)    |                               |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Lemlist - Lead Replied Node**  
   - Type: lemlistTrigger  
   - Event: "emailsReplied"  
   - Options: Set `isFirst` to true to trigger only on first reply  
   - Credentials: Configure with lemlist API key (team API)  
   - Position: Left side of canvas  

2. **Create OpenAI Node**  
   - Type: openAi  
   - Prompt:  
     ```
     The following is a list of emails and the categories they fall into:
     Categories=["interested", "Out of office", "unsubscribe", "other"]

     Interested is when the reply is positive.

     "{{$json["text"].replaceAll(/^\s+|\s+$/g, '').replace(/(\r\n|\n|\r)/gm, "")}}"
     Category:
     ```  
   - Parameters: temperature=0, maxTokens=6, topP=1  
   - Credentials: Configure with OpenAI API key  
   - Position: Right of Lemlist - Lead Replied  

3. **Create Merge Node**  
   - Type: merge  
   - Mode: Combine  
   - Combination Mode: Merge By Position  
   - Clash Handling: Prefer Input 1 values  
   - Inputs: Two inputs from Lemlist - Lead Replied and OpenAI nodes  
   - Position: Between OpenAI and Switch node  

4. **Create Switch Node**  
   - Type: switch  
   - Rules:  
     - If trimmed classification text equals "Unsubscribe" → Output 0  
     - If equals "Interested" → Output 1  
     - If equals "Out of Office" → Output 2  
     - Fallback → Output 3  
   - Expression for value1: `={{ $json["text"].trim() }}`  
   - Position: Right of Merge node  

5. **Create Lemlist - Unsubscribe Node**  
   - Type: lemlist  
   - Resource: lead  
   - Operation: unsubscribe  
   - Email: `={{ $json["leadEmail"] }}`  
   - Campaign ID: `={{ $json["campaignId"] }}`  
   - Credentials: lemlist API key  
   - Position: Output 0 branch from Switch (Unsubscribe)  

6. **Create lemlist - Mark as interested Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.lemlist.com/api/campaigns/YOUR_CAMPAIGN_ID/leads/{{$json["leadEmail"]}}/interested`  
   - Credentials: lemlist API key  
   - Position: Output 1 branch from Switch (Interested)  

7. **Create HubSpot - Get contact ID Node**  
   - Type: hubspot  
   - Resource: contact  
   - Email: `={{ $json["leadEmail"] }}`  
   - Additional fields:  
     - firstName: `={{ $json["leadFirstName"] }}`  
     - lastName: `={{ $json["leadLastName"] }}`  
   - Authentication: OAuth2  
   - Credentials: HubSpot account OAuth2  
   - Position: Parallel to lemlist - Mark as interested, connected from Switch output 1  

8. **Connect lemlist - Mark as interested and HubSpot - Get contact ID to HubSpot - Create Deal**  
   - Create HubSpot - Create Deal Node:  
     - Stage: "79009480" (replace with your pipeline stage ID)  
     - Deal Name: `=New Deal with {{ $json["identity-profiles"][0]["identities"][0]["value"] }}`  
     - Associated VIDs: `={{ $json["canonical-vid"] }}`  
     - Credentials: HubSpot OAuth2  
   - Position: Connected from HubSpot - Get contact ID output  

9. **Create Slack Node for Interested Leads**  
   - Type: slack  
   - Text:  
     ```
     Hello a new lead is interested.

     More info in Hubspot here: 
     https://app-eu1.hubspot.com/contacts/25897606/deal/{{$json["dealId"]}}
     ```  
   - Channel: Your Slack channel name (replace accordingly)  
   - Credentials: Slack OAuth2  
   - Position: Connected from HubSpot - Create Deal output  

10. **Create HubSpot - Get contact ID1 Node for OOO leads**  
    - Same setup as HubSpot - Get contact ID node but used for Out of Office leads  
    - Connected from Switch output 2 (Out of Office)  

11. **Create follow up task Node**  
    - Type: hubspot  
    - Resource: engagement (task)  
    - Subject: `=OOO - Follow up with {{ $json["properties"]["firstname"]["value"] }} {{ $json["properties"]["lastname"]["value"] }}`  
    - Associations: contactIds: `={{ $json["vid"] }}`  
    - Credentials: HubSpot OAuth2  
    - Position: Connected from HubSpot - Get contact ID1 output  

12. **Create Slack1 Node for Fallback Category**  
    - Type: slack  
    - Text:  
      ```
      Hello a lead replied to your emails.

      More info in lemlist here: 
      https://app.lemlist.com/teams/{{$json["teamId"]}}/reports/campaigns/{{$json["campaignId"]}}
      ```  
    - Channel: Your Slack channel name (replace accordingly)  
    - Credentials: Slack OAuth2  
    - Position: Connected from Switch output 3 (Fallback)  

13. **Connect all nodes according to the flow described in the Overview section.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The lemlist "Mark as interested" node uses a hardcoded campaign ID `YOUR_CAMPAIGN_ID` that must be replaced with your actual campaign ID. | Critical for correct API operation in production.                                                               |
| Slack messages contain URLs to HubSpot and lemlist dashboards which include dynamic IDs; update these to fit your workspace and campaigns. | Ensure URLs point to the correct instances and teams to avoid broken links.                                     |
| HubSpot nodes require OAuth2 credentials with sufficient scopes for contacts, deals, and engagements management. | Credential setup must be verified before running workflow.                                                     |
| OpenAI node prompt is designed for deterministic classification; changing temperature or max tokens may affect classification reliability. | Adjust only if you understand the impact on GPT-3 output consistency.                                           |
| Webhook ID in Lemlist trigger node must match your actual webhook configuration in lemlist dashboard.       | Misconfiguration will cause trigger failures.                                                                   |

---

This detailed reference document enables understanding, modification, error anticipation, and full reconstruction of the "lemlist <> GPT-3: Supercharge your sales workflows" n8n automation.