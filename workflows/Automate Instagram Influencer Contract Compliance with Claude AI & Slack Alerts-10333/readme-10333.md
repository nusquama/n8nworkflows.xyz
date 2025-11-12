Automate Instagram Influencer Contract Compliance with Claude AI & Slack Alerts

https://n8nworkflows.xyz/workflows/automate-instagram-influencer-contract-compliance-with-claude-ai---slack-alerts-10333


# Automate Instagram Influencer Contract Compliance with Claude AI & Slack Alerts

### 1. Workflow Overview

This workflow automates monitoring Instagram influencer contracts to ensure compliance with deliverables and deadlines. It periodically checks contract deadlines, sends reminders for approaching deadlines, verifies influencer posts against contract requirements using Claude AI, and alerts managers via Slack if any breaches occur. The workflow is designed primarily for marketing teams managing influencer campaigns, ensuring timely posts with required hashtags and campaign-specific content.

Logical blocks included:

- **1.1 Trigger and Contract Retrieval:** Scheduled or webhook-based initiation and loading influencer contracts from Google Sheets.
- **1.2 Contract Processing Loop:** Iterates over each contract individually for compliance validation.
- **1.3 Deadline Calculation and Branching:** Calculates days remaining until deadline and branches on approaching or passed deadlines.
- **1.4 Reminder Notifications:** Sends Slack reminders for contracts with deadlines approaching within 3 days.
- **1.5 Post Retrieval and AI Compliance Check:** Fetches recent Instagram posts of the influencer and uses Claude AI to verify contract compliance.
- **1.6 Breach Handling:** If non-compliance is detected, alerts managers via Slack, logs the breach in Google Sheets, and responds to webhook calls.
- **1.7 Compliance End Path:** Terminates workflow branches where contracts are compliant, ensuring no further action.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Contract Retrieval

- **Overview:** Initiates the workflow either on a scheduled basis (daily at 9 AM) or via webhook, then loads influencer contracts from a Google Sheets document.
- **Nodes Involved:** Schedule Trigger, Webhook, Get Contracts
- **Node Details:**

  - **Schedule Trigger**
    - Type: Cron Trigger
    - Config: Runs daily at 9 AM (implicit from sticky note)
    - Input: None (time-based trigger)
    - Output: Triggers "Get Contracts"
    - Edge Cases: Cron misconfiguration could stop runs; time zone considerations.

  - **Webhook**
    - Type: Webhook Trigger
    - Config: Listens on path `/f50af7ff-5c16-4985-9f9f-d10b805b11d4`
    - Input: External HTTP requests triggering the workflow
    - Output: Triggers "Get Contracts"
    - Edge Cases: Unauthorized external calls if webhook is public; missing parameters.

  - **Get Contracts**
    - Type: Google Sheets Read
    - Config: Reads range `A:H` from specified Google Sheet containing contract data
    - Credentials: Google API credentials configured
    - Input: Trigger from Schedule Trigger or Webhook
    - Output: Contract data array to "Loop Over Contracts"
    - Edge Cases: Sheet ID misconfiguration, API auth errors, sheet range errors, empty sheets
    - Sticky Note: "Loads influencer contracts from Contracts sheet"

#### 2.2 Contract Processing Loop

- **Overview:** Processes each contract record individually in batches of one to handle contracts sequentially.
- **Nodes Involved:** Loop Over Contracts
- **Node Details:**

  - **Loop Over Contracts**
    - Type: SplitInBatches
    - Config: Batch size = 1 (process one contract at a time)
    - Input: Contract array from "Get Contracts"
    - Output: Single contract JSON to "Calculate Deadline Status"
    - Edge Cases: Large number of contracts may increase execution time; batch size too large could cause memory issues
    - Sticky Note: "Processes each contract individually"

#### 2.3 Deadline Calculation and Branching

- **Overview:** Calculates days remaining till contract deadline, flags contracts as approaching deadline (within 3 days) or passed deadline, then branches workflow accordingly.
- **Nodes Involved:** Calculate Deadline Status, IF Approaching Deadline, IF Deadline Passed
- **Node Details:**

  - **Calculate Deadline Status**
    - Type: Code (JavaScript)
    - Config: Computes days to deadline by subtracting current date from the contract's deadline date; sets flags:
      - `isApproaching`: true if 1–3 days remaining
      - `isPassed`: true if deadline passed or today
    - Input: Single contract JSON
    - Output: Enhanced JSON with deadline status to IF nodes
    - Edge Cases: Invalid or missing date format causes errors; time zone issues may affect calculation
    - Sticky Note: "Computes days to deadline, flags approaching/passed"

  - **IF Approaching Deadline**
    - Type: If
    - Config: Checks if `isApproaching` is true
    - Input: Contract with deadline status
    - Output (true): "Send Reminder (Slack)"
    - Output (false): Continues to "IF Deadline Passed"
    - Edge Cases: Expression evaluation failure if field missing
    - Sticky Note: "Branches if within 3 days"

  - **IF Deadline Passed**
    - Type: If
    - Config: Checks if `isPassed` is true
    - Input: Contract with deadline status
    - Output (true): "Get Influencer Posts"
    - Output (false): End of contract processing (no further nodes)
    - Edge Cases: Similar to above; missing or invalid flag
    - Sticky Note: "Branches if deadline has passed"

#### 2.4 Reminder Notifications

- **Overview:** Sends a Slack reminder message for contracts approaching their deadlines within 3 days.
- **Nodes Involved:** Send Reminder (Slack)
- **Node Details:**

  - **Send Reminder (Slack)**
    - Type: Slack Node (Message)
    - Config: Posts a reminder message to a configured Slack channel including influencer name, days to deadline, campaign ID, and required hashtag.
    - Credentials: Slack OAuth2 credentials configured
    - Input: Contract JSON flagged as approaching deadline
    - Output: None (endpoint)
    - Edge Cases: Slack API failures, credential expiration, channel misconfiguration
    - Sticky Note: "Notifies influencer/team of upcoming deadline"

#### 2.5 Post Retrieval and AI Compliance Check

- **Overview:** For contracts past due, fetches recent Instagram posts of the influencer, then uses Claude AI to verify if posts meet contract requirements, including required hashtags and campaign description.
- **Nodes Involved:** Get Influencer Posts, Check Post Compliance (Claude AI), IF Breach Detected
- **Node Details:**

  - **Get Influencer Posts**
    - Type: HTTP Request
    - Config: Calls Facebook Graph API endpoint `/v12.0/{influencerUserId}/media` using an Instagram access token to retrieve recent posts.
    - Input: Contract JSON with influencer user ID
    - Output: Instagram post data to AI check
    - Edge Cases: API token expiration, rate limits, influencer ID invalid, HTTP failures
    - Sticky Note: "Fetches recent posts from influencer’s Instagram"

  - **Check Post Compliance (Claude AI)**
    - Type: HTTP Request (POST)
    - Config: Calls Anthropic Claude AI API with model `claude-3-5-sonnet-20241022`. Sends prompt querying if recent posts contain required hashtag and match campaign description, expecting a yes/no response.
    - Input: Instagram posts and contract requirements embedded in prompt JSON
    - Output: AI response JSON including compliance indicator
    - Edge Cases: API key/auth errors, rate limits, malformed requests, unexpected response format
    - Sticky Note: "Uses AI to verify if post matches contract requirements"

  - **IF Breach Detected**
    - Type: If
    - Config: Parses AI response content text. Checks if compliance answer is "no" (case-insensitive, trimmed).
    - Input: AI compliance check JSON
    - Output (true): "Notify Breach (Slack)", "Log Breach", "Respond to Webhook"
    - Output (false): "End (Compliant Path)"
    - Edge Cases: Parsing errors if AI response format changes, false positives/negatives
    - Sticky Note: "Branches if non-compliant"

#### 2.6 Breach Handling

- **Overview:** Alerts managers on Slack about contract breaches, logs breach details into Google Sheets, and responds to webhook calls if applicable.
- **Nodes Involved:** Notify Breach (Slack), Log Breach, Respond to Webhook
- **Node Details:**

  - **Notify Breach (Slack)**
    - Type: Slack Node (Message)
    - Config: Sends alert message to a manager Slack channel with influencer name, campaign ID, and missed deadline.
    - Credentials: Slack OAuth2 credentials configured
    - Input: Contract JSON with breach detected
    - Output: None
    - Edge Cases: Slack API failures, channel misconfiguration
    - Sticky Note: "Alerts manager of contract breach"

  - **Log Breach**
    - Type: Google Sheets Append
    - Config: Appends breach details (columns A:F) to a Logs sheet for record-keeping.
    - Credentials: Google API credentials configured
    - Input: Breach JSON data
    - Output: None
    - Edge Cases: Sheet permission issues, API rate limits
    - Sticky Note: "Records breach details in Logs sheet"

  - **Respond to Webhook**
    - Type: Respond to Webhook
    - Config: Sends HTTP response to webhook caller, confirming workflow completion.
    - Input: From breach detection path
    - Output: Ends workflow execution
    - Edge Cases: Webhook not triggered, multiple simultaneous requests
    - Sticky Note: "Terminates compliant branches"

#### 2.7 Compliance End Path

- **Overview:** Ends workflow branches where contracts are compliant or no issues are detected.
- **Nodes Involved:** End (Compliant Path)
- **Node Details:**

  - **End (Compliant Path)**
    - Type: Set
    - Config: Empty node used to explicitly end branches with compliant contracts.
    - Input: Contracts passing AI compliance checks
    - Output: None
    - Edge Cases: None
    - Sticky Note: Combined with breach handling sticky note "Terminates compliant branches"

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                           | Input Node(s)                | Output Node(s)                             | Sticky Note                                                   |
|-----------------------------|------------------------|-----------------------------------------|-----------------------------|-------------------------------------------|---------------------------------------------------------------|
| Schedule Trigger             | Cron Trigger           | Initiates workflow daily at 9 AM        | None                        | Get Contracts                             | Runs daily at 9 AM or via webhook (/influencer-compliance)    |
| Webhook                     | Webhook Trigger        | Initiates workflow on HTTP request      | None                        | Get Contracts                             | Runs daily at 9 AM or via webhook (/influencer-compliance)    |
| Get Contracts               | Google Sheets          | Loads influencer contracts from sheet   | Schedule Trigger, Webhook   | Loop Over Contracts                       | Loads influencer contracts from Contracts sheet               |
| Loop Over Contracts         | SplitInBatches         | Processes contracts individually         | Get Contracts               | Calculate Deadline Status                 | Processes each contract individually                           |
| Calculate Deadline Status   | Code (JavaScript)      | Calculates days to deadline, flags      | Loop Over Contracts         | IF Approaching Deadline, IF Deadline Passed | Computes days to deadline, flags approaching/passed           |
| IF Approaching Deadline     | If                     | Checks if deadline is approaching        | Calculate Deadline Status   | Send Reminder (Slack), IF Deadline Passed | Branches if within 3 days                                      |
| Send Reminder (Slack)       | Slack                  | Sends Slack reminders for approaching deadlines | IF Approaching Deadline    | None                                      | Notifies influencer/team of upcoming deadline                  |
| IF Deadline Passed          | If                     | Checks if deadline has passed             | Calculate Deadline Status   | Get Influencer Posts                      | Branches if deadline has passed                                |
| Get Influencer Posts        | HTTP Request           | Fetches recent Instagram posts            | IF Deadline Passed          | Check Post Compliance (Claude AI)        | Fetches recent posts from influencer’s Instagram              |
| Check Post Compliance (Claude AI) | HTTP Request     | Uses AI to verify contract compliance     | Get Influencer Posts        | IF Breach Detected                        | Uses AI to verify if post matches contract requirements       |
| IF Breach Detected          | If                     | Checks AI compliance result               | Check Post Compliance       | Notify Breach (Slack), Log Breach, Respond to Webhook; End (Compliant Path) | Branches if non-compliant                                     |
| Notify Breach (Slack)       | Slack                  | Alerts managers about contract breach    | IF Breach Detected          | None                                      | Alerts manager of contract breach                             |
| Log Breach                  | Google Sheets          | Logs breach details in sheet              | IF Breach Detected          | None                                      | Records breach details in Logs sheet                          |
| Respond to Webhook          | Respond to Webhook     | Sends response to webhook caller          | IF Breach Detected          | None                                      | Terminates compliant branches                                 |
| End (Compliant Path)        | Set                    | Ends compliant workflow branches          | IF Breach Detected          | None                                      | Terminates compliant branches                                 |
| Sticky Note                 | Sticky Note            | Visual note                              | N/A                         | N/A                                       | See detailed sticky notes in node descriptions                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Cron Trigger** node named "Schedule Trigger":
     - Set to run daily at 9 AM (adjust timezone as needed).

   - Add a **Webhook** node named "Webhook":
     - Set path to a unique identifier (e.g., `/f50af7ff-5c16-4985-9f9f-d10b805b11d4`).
     - Leave options default.

2. **Create Contract Retrieval Node:**

   - Add **Google Sheets** node named "Get Contracts":
     - Operation: Read
     - Sheet ID: Specify your contracts sheet and range (e.g., `your-contracts-sheet-id!A:H`).
     - Credentials: Configure Google API credentials with access to the sheet.

3. **Connect Triggers to Contract Retrieval:**

   - Connect "Schedule Trigger" → "Get Contracts"
   - Connect "Webhook" → "Get Contracts"

4. **Create Contract Processing Loop:**

   - Add **SplitInBatches** node named "Loop Over Contracts":
     - Batch Size: 1
     - Connect "Get Contracts" → "Loop Over Contracts"

5. **Add Deadline Calculation Node:**

   - Add **Code** node named "Calculate Deadline Status":
     - Language: JavaScript
     - Code:
       ```javascript
       const deadline = new Date($json['deadline']);
       const now = new Date();
       const daysToDeadline = Math.ceil((deadline - now) / (1000 * 60 * 60 * 24));
       
       return {
         json: {
           ...$json,
           daysToDeadline,
           isApproaching: daysToDeadline > 0 && daysToDeadline <= 3,
           isPassed: daysToDeadline <= 0
         }
       };
       ```
     - Connect "Loop Over Contracts" → "Calculate Deadline Status"

6. **Add Branching Nodes for Deadline Status:**

   - Add **If** node "IF Approaching Deadline":
     - Condition: Boolean | Value1: `{{$json["isApproaching"]}}` | Value2: `true`
     - Connect "Calculate Deadline Status" → "IF Approaching Deadline"

   - Add **If** node "IF Deadline Passed":
     - Condition: Boolean | Value1: `{{$json["isPassed"]}}` | Value2: `true`
     - Connect "Calculate Deadline Status" → "IF Deadline Passed"

7. **Add Reminder Notification Node:**

   - Add **Slack** node "Send Reminder (Slack)":
     - Channel: Your Slack reminder channel
     - Text:
       ```
       Reminder: Influencer {{$json["influencerName"]}} - Post due in {{$json["daysToDeadline"]}} days for campaign {{$json["campaignId"]}}. Required hashtag: {{$json["requiredHashtag"]}}
       ```
     - Credentials: Configure Slack OAuth2 credentials
     - Connect "IF Approaching Deadline" (true) → "Send Reminder (Slack)"

   - Connect "IF Approaching Deadline" (false) → "IF Deadline Passed"

8. **Add Instagram Post Retrieval Node:**

   - Add **HTTP Request** node "Get Influencer Posts":
     - Method: GET
     - URL:
       ```
       https://graph.facebook.com/v12.0/{{$json["influencerUserId"]}}/media?access_token=YOUR_INSTAGRAM_ACCESS_TOKEN
       ```
     - Connect "IF Deadline Passed" (true) → "Get Influencer Posts"

9. **Add Claude AI Compliance Check Node:**

   - Add **HTTP Request** node "Check Post Compliance (Claude AI)":
     - Method: POST
     - URL: `https://api.anthropic.com/v1/messages`
     - Authentication: Set API Key for Anthropic Claude
     - Body Parameters (JSON):
       ```json
       {
         "model": "claude-3-5-sonnet-20241022",
         "max_tokens": 1024,
         "messages": [
           {
             "role": "user",
             "content": "Check if any of these recent Instagram posts contain the required hashtag '{{ $json[\"requiredHashtag\"] }}' and match the campaign description '{{ $json[\"campaignDescription\"] }}'. Posts: {{ JSON.stringify($json[\"data\"]) }}. Output: is_compliant: yes/no"
           }
         ]
       }
       ```
     - Enable JSON parameters
     - Connect "Get Influencer Posts" → "Check Post Compliance (Claude AI)"

10. **Add Breach Detection Node:**

    - Add **If** node "IF Breach Detected":
      - Condition: String
      - Value1 (Expression):
        ```
        {{$json["content"][0]["text"].toLowerCase().split(':')[1].trim()}}
        ```
      - Value2: `no`
      - Connect "Check Post Compliance (Claude AI)" → "IF Breach Detected"

11. **Add Breach Notification and Logging Nodes:**

    - Add **Slack** node "Notify Breach (Slack)":
      - Channel: Manager Slack channel
      - Text:
        ```
        Contract Breach Alert! Influencer {{$json["influencerName"]}} failed to post for campaign {{$json["campaignId"]}} by deadline {{$json["deadline"]}}.
        ```
      - Credentials: Slack OAuth2 credentials
      - Connect "IF Breach Detected" (true) → "Notify Breach (Slack)"

    - Add **Google Sheets** node "Log Breach":
      - Operation: Append
      - Sheet ID: Your logs sheet (`your-logs-sheet-id!A:F`)
      - Credentials: Google API credentials
      - Connect "Notify Breach (Slack)" → "Log Breach"

    - Add **Respond to Webhook** node:
      - Connect "Log Breach" → "Respond to Webhook"

12. **Add Compliant End Node:**

    - Add **Set** node "End (Compliant Path)":
      - No parameters; used to end branch explicitly
      - Connect "IF Breach Detected" (false) → "End (Compliant Path)"

    - Also connect "IF Approaching Deadline" (false) → "IF Deadline Passed" (false) path to this end node or leave implicit.

13. **Final Connections:**

    - Ensure "Send Reminder (Slack)" node has no outputs (ends there).
    - "Respond to Webhook" ends workflow on breach.
    - "End (Compliant Path)" ends compliant contract processing branches.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow runs daily at 9 AM or can be triggered manually via webhook `/influencer-compliance`.                                               | Sticky Note on Schedule Trigger and Webhook                  |
| Requires Google Sheets with influencer contracts and logs sheets properly formatted and accessible with API credentials.                   | Google Sheets node configuration                              |
| Instagram Graph API access token must have permissions to read influencer media. Token expiration and rate limits must be monitored.        | Get Influencer Posts node                                     |
| Slack OAuth2 credentials required for reminder and breach alert notifications. Channels must be configured accordingly.                     | Slack nodes                                                  |
| Claude AI API key with access to `claude-3-5-sonnet-20241022` model required. Prompt design critical for accurate compliance checks.       | Check Post Compliance (Claude AI) node                       |
| Use careful error handling for date parsing and API calls to avoid workflow crashes or false alerts. Consider adding retry or error nodes. | General best practice                                        |
| Workflow credit: n8n automation integration for influencer compliance monitoring.                                                            | Project purpose                                               |

---

This structured reference document enables clear understanding, reproduction, and modification of the complete Instagram Influencer Contract Compliance automation workflow using Claude AI and Slack integration.