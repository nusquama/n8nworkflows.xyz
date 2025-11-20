Send or update new Mailchimp subscribers in HubSpot

https://n8nworkflows.xyz/workflows/send-or-update-new-mailchimp-subscribers-in-hubspot-1771


# Send or update new Mailchimp subscribers in HubSpot

### 1. Workflow Overview

This workflow automates the synchronization of new or updated Mailchimp subscribers into HubSpot as contacts. It is designed for daily operation to keep HubSpot contact lists up to date with the latest Mailchimp subscriber data, either by creating new contacts or updating existing ones.

Logical blocks:

- **1.1 Scheduling & State Tracking:** Manages periodic triggering and tracks the last execution timestamp to query only new or changed Mailchimp subscribers.
- **1.2 Mailchimp Data Retrieval:** Fetches subscriber updates from Mailchimp since the last workflow run.
- **1.3 HubSpot Contact Management:** Creates or updates contacts in HubSpot using the subscriber data from Mailchimp.
- **1.4 Update Execution Timestamp:** Updates the stored timestamp to mark the current run completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & State Tracking

- **Overview:** Triggers the workflow daily at 7:00 AM and retrieves the last execution time from static workflow data to query incremental subscriber changes.
- **Nodes Involved:**  
  - Every day at 07:00 (Cron)  
  - Get last execution timestamp (FunctionItem)

- **Node Details:**

  - **Every day at 07:00**  
    - *Type:* Cron trigger  
    - *Configuration:* Triggers once daily at 7:00 AM (hour set to 7, no minutes specified defaults to 0)  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Triggers next node  
    - *Edge cases:* Cron misfire if server time changes; ensure server timezone matches expected schedule.

  - **Get last execution timestamp**  
    - *Type:* FunctionItem (runs once per input item)  
    - *Configuration:*  
      - Retrieves a global static data key `lastExecution`. If not set, initializes it with current date/time.  
      - Adds two fields to the item output:  
        - `executionTimeStamp`: current date/time  
        - `lastExecution`: the stored timestamp from last run  
    - *Key expressions:* Uses `getWorkflowStaticData('global')` for persistent storage  
    - *Input:* Trigger from Cron node  
    - *Output:* Passes timestamp data to next node  
    - *Edge cases:* Static data missing or inaccessible; first run sets lastExecution to current time, which may cause no subscribers fetched on first run.

#### 1.2 Mailchimp Data Retrieval

- **Overview:** Queries Mailchimp for all subscribers changed since the last workflow execution timestamp.
- **Nodes Involved:**  
  - Get changed members (Mailchimp)

- **Node Details:**

  - **Get changed members**  
    - *Type:* Mailchimp node  
    - *Operation:* `getAll` to retrieve all members from a specified list who changed since a given timestamp  
    - *Configuration:*  
      - List ID hardcoded as `"bcfb6ff8f1"` (must be replaced with actual list ID)  
      - Option `sinceLastChanged` set dynamically from previous node output: `={{ $json["lastExecution"] }}`  
    - *Credentials:* Mailchimp API credentials required and configured  
    - *Input:* Receives timestamp data from previous function node  
    - *Output:* Outputs an array of subscriber objects with fields like `email_address`, `merge_fields` (which contains first and last names)  
    - *Edge cases:*  
      - Invalid or expired Mailchimp credentials cause auth errors  
      - No changes since last execution results in empty output  
      - Incorrect list ID causes API errors

#### 1.3 HubSpot Contact Management

- **Overview:** For each subscriber received from Mailchimp, creates or updates a contact in HubSpot with matching email and name fields.
- **Nodes Involved:**  
  - Create/Update contact (HubSpot)

- **Node Details:**

  - **Create/Update contact**  
    - *Type:* HubSpot node  
    - *Resource:* Contact  
    - *Authentication:* App Token method (API token)  
    - *Configuration:*  
      - Email mapped from `email_address` field of Mailchimp subscriber  
      - First and last names mapped from `merge_fields.FNAME` and `merge_fields.LNAME` respectively  
    - *Credentials:* HubSpot App Token credentials configured  
    - *Input:* Receives array of subscribers from Mailchimp node  
    - *Output:* Passes contact creation/update result to next node  
    - *Edge cases:*  
      - Invalid HubSpot token causes authentication failure  
      - Missing email or malformed email address may cause API rejection  
      - HubSpot API rate limits or downtime

#### 1.4 Update Execution Timestamp

- **Overview:** Updates the global static data `lastExecution` with the current execution timestamp to ensure the workflow queries Mailchimp for changes only after this time in future runs.
- **Nodes Involved:**  
  - Set new last execution timestamp (FunctionItem)

- **Node Details:**

  - **Set new last execution timestamp**  
    - *Type:* FunctionItem  
    - *Configuration:*  
      - Sets global static data `lastExecution` to the `executionTimeStamp` recorded at the start of this run (taken from the output of “Get last execution timestamp” node)  
    - *Input:* Receives the output from HubSpot contact update node  
    - *Output:* None needed beyond workflow end  
    - *Execute Once:* Enabled to run only once for the entire input set  
    - *Edge cases:* Failure to update static data leads to repeated processing of old data on next run

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                      | Input Node(s)            | Output Node(s)                | Sticky Note                          |
|-----------------------------|---------------------|------------------------------------|--------------------------|------------------------------|------------------------------------|
| Every day at 07:00           | Cron                | Trigger workflow daily at 7:00 AM  | None                     | Get last execution timestamp  |                                    |
| Get last execution timestamp | FunctionItem        | Retrieve last run timestamp or init| Every day at 07:00       | Get changed members           |                                    |
| Get changed members          | Mailchimp           | Fetch subscribers changed since last run | Get last execution timestamp | Create/Update contact          | Mailchimp credentials required: https://docs.n8n.io/integrations/builtin/credentials/mailchimp/ |
| Create/Update contact        | HubSpot             | Create or update contact in HubSpot| Get changed members       | Set new last execution timestamp | HubSpot credentials required: https://docs.n8n.io/integrations/builtin/credentials/hubspot/ |
| Set new last execution timestamp | FunctionItem  | Update last execution timestamp    | Create/Update contact     | None                         |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node:**  
   - Add node type: Cron  
   - Name it: `Every day at 07:00`  
   - Set trigger time: hour = 7, minute = 0 (default)  
   - No credentials needed

2. **Create FunctionItem Node:**  
   - Name: `Get last execution timestamp`  
   - Paste the following code in the function field:
     ```javascript
     const staticData = getWorkflowStaticData('global');
     if(!staticData.lastExecution){
       staticData.lastExecution = new Date();
     }
     item.executionTimeStamp = new Date();
     item.lastExecution = staticData.lastExecution;
     return item;
     ```
   - Connect output of `Every day at 07:00` to this node

3. **Create Mailchimp Node:**  
   - Name: `Get changed members`  
   - Set resource to `Members` or equivalent  
   - Operation: `getAll`  
   - Set parameter `List` to your Mailchimp list ID (e.g., `"bcfb6ff8f1"`)  
   - Under options, set `Since Last Changed` parameter to: `={{ $json["lastExecution"] }}` (expression mode)  
   - Assign Mailchimp credentials (create or select existing)  
   - Connect output of `Get last execution timestamp` to this node

4. **Create HubSpot Node:**  
   - Name: `Create/Update contact`  
   - Resource: Contact  
   - Operation: Create or update contact (default for contact resource)  
   - Set Email field to: `={{ $json["email_address"] }}`  
   - Set Additional Fields:  
     - First Name: `={{ $json["merge_fields"].FNAME }}`  
     - Last Name: `={{ $json["merge_fields"].LNAME }}`  
   - Configure HubSpot App Token credentials  
   - Connect output of `Get changed members` to this node

5. **Create FunctionItem Node:**  
   - Name: `Set new last execution timestamp`  
   - Paste the following code:
     ```javascript
     const staticData = getWorkflowStaticData('global');
     staticData.lastExecution = $item(0).$node["Get last execution timestamp"].executionTimeStamp;
     return item;
     ```
   - Enable "Execute Once" option  
   - Connect output of `Create/Update contact` to this node

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Mailchimp credentials setup is required for the Mailchimp node to authenticate API requests. See official docs for setup details. | https://docs.n8n.io/integrations/builtin/credentials/mailchimp/ |
| HubSpot credentials using App Token method are required to authorize contact creation/update operations. | https://docs.n8n.io/integrations/builtin/credentials/hubspot/ |
| The static workflow data storage is used to persist the last execution timestamp across runs to support incremental data fetching. | n8n documentation on static data usage |
| The workflow assumes consistent server timezone aligned with the expected daily trigger time. Adjust cron node timezone if needed. | n8n Cron node documentation |
| The Mailchimp list ID must be replaced with your actual list ID. You can find this in your Mailchimp account under audience settings. | Mailchimp list ID info |

---

This document fully describes the workflow’s design, node configurations, and operational logic to enable accurate reproduction, modification, and troubleshooting.