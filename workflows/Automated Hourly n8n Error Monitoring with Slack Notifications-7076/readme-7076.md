Automated Hourly n8n Error Monitoring with Slack Notifications

https://n8nworkflows.xyz/workflows/automated-hourly-n8n-error-monitoring-with-slack-notifications-7076


# Automated Hourly n8n Error Monitoring with Slack Notifications

### 1. Workflow Overview

This workflow automates hourly monitoring of n8n workflow executions to detect failures and notify a Slack channel with detailed alerts. It targets n8n administrators and DevOps teams who need real-time visibility into workflow errors without manual checking. The workflow logically divides into these blocks:

- **1.1 Scheduling and Configuration:** Periodically triggers execution and sets up base configuration data (e.g., URL prefixes).
- **1.2 Workflow Retrieval:** Fetches all active workflows to check for execution errors.
- **1.3 Iterative Error Querying:** Processes each workflow individually to query recent failed executions.
- **1.4 Aggregation and Filtering:** Aggregates error data and filters for errors occurring within the last hour.
- **1.5 Message Construction:** Builds Slack message blocks summarizing error counts and links.
- **1.6 Notification Dispatch:** Sends the constructed message to a specified Slack user via Slack API.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Configuration

- **Overview:** This block defines the hourly rhythm of the workflow and prepares base variables like URL prefixes needed later for error report links.
- **Nodes Involved:** Schedule Trigger, Config
- **Node Details:**

  - **Schedule Trigger**
    - *Type:* Trigger node
    - *Role:* Initiates the workflow every hour.
    - *Configuration:* Interval set to every 1 hour.
    - *Inputs:* None (trigger node).
    - *Outputs:* Triggers Config node.
    - *Potential Failures:* Scheduling misconfiguration or system clock issues.

  - **Config**
    - *Type:* Set node
    - *Role:* Sets a static JSON variable `urlBase` to prefix workflow URLs.
    - *Configuration:* Raw JSON output with `"urlBase": "https://n8n.com/workflow/"`.
    - *Inputs:* Schedule Trigger.
    - *Outputs:* Triggers GetWorkflows node.
    - *Potential Failures:* None significant; static data.

#### 2.2 Workflow Retrieval

- **Overview:** Retrieves the list of all active n8n workflows to be checked for errors.
- **Nodes Involved:** GetWorkflows
- **Node Details:**

  - **GetWorkflows**
    - *Type:* n8n API node
    - *Role:* Queries active workflows via n8n internal API.
    - *Configuration:* Filter set to only active workflows.
    - *Credentials:* Uses n8n API credentials configured externally.
    - *Inputs:* Config node.
    - *Outputs:* Sends list of workflows to Loop node.
    - *Potential Failures:* API authentication failure, connectivity issues, empty workflow list.

#### 2.3 Iterative Error Querying

- **Overview:** Iterates through each active workflow, queries the latest failed execution (if any) for that workflow.
- **Nodes Involved:** Loop, n8n
- **Node Details:**

  - **Loop**
    - *Type:* SplitInBatches
    - *Role:* Processes workflows one at a time to avoid API overload.
    - *Configuration:* Default batching options (no batch size specified, defaults to 1).
    - *Inputs:* GetWorkflows node.
    - *Outputs:* Feeds each workflow into two branches: n8n and MakeMessage nodes.
    - *Potential Failures:* Large workflow counts could impact performance.

  - **n8n**
    - *Type:* n8n API node
    - *Role:* Queries the most recent failed execution for the current workflow.
    - *Configuration:* Filters for executions with status "error" limited to 1 result, filtered by current workflow ID from Loop node.
    - *Credentials:* Same n8n API credentials.
    - *Inputs:* Loop node.
    - *Outputs:* Aggregated with other results in Aggregate node.
    - *Potential Failures:* API rate limit, authentication failure, empty error executions.

#### 2.4 Aggregation and Filtering

- **Overview:** Aggregates all error execution data for all workflows and filters to retain only those failed executions from the last hour.
- **Nodes Involved:** Aggregate, FilterLastHour
- **Node Details:**

  - **Aggregate**
    - *Type:* Aggregate node
    - *Role:* Combines all items from n8n node executions into a single output array.
    - *Configuration:* Aggregates all item data.
    - *Inputs:* n8n node.
    - *Outputs:* Passes combined data to FilterLastHour node.
    - *Potential Failures:* Handling empty input arrays gracefully.

  - **FilterLastHour**
    - *Type:* Code node (JavaScript)
    - *Role:* Filters the aggregated execution errors to keep only those stopped within the last hour.
    - *Configuration:* 
      - Reads aggregated data from Aggregate node.
      - Compares each execution’s `stoppedAt` timestamp against current time minus one hour.
      - Returns a summary object with workflow ID, name, count of errors in last hour, and URL to workflow.
    - *Inputs:* Aggregate node.
    - *Outputs:* Sends filtered data back to Loop node for message preparation.
    - *Edge Cases:* Timezone issues, empty input arrays, date parsing errors.

#### 2.5 Message Construction

- **Overview:** Builds Slack message blocks listing workflows with error counts and links for visualization.
- **Nodes Involved:** MakeMessage
- **Node Details:**

  - **MakeMessage**
    - *Type:* Code node (JavaScript)
    - *Role:* Constructs a Slack message payload using Slack Block Kit format.
    - *Configuration:*
      - Maps over all filtered workflow error summaries.
      - For each workflow with errors, creates a section block with workflow name, error count, and a "Visualize" button linking to the workflow URL.
      - Returns the blocks array or null if no errors found.
    - *Inputs:* Loop node (aggregated filtered data).
    - *Outputs:* Slack node.
    - *Edge Cases:* No errors found (returns null, preventing Slack message), malformed URLs.

#### 2.6 Notification Dispatch

- **Overview:** Sends the constructed Slack message blocks to a configured Slack user.
- **Nodes Involved:** Slack
- **Node Details:**

  - **Slack**
    - *Type:* Slack node
    - *Role:* Sends a formatted message to a Slack user via OAuth2 authentication.
    - *Configuration:*
      - Message type set to block.
      - User set by ID `U05FMSA2610`.
      - Blocks content dynamically populated from MakeMessage node output.
    - *Credentials:* Slack OAuth2 API credentials configured externally.
    - *Inputs:* MakeMessage node.
    - *Potential Failures:* Slack API rate limits, authentication errors, invalid user ID.

---

### 3. Summary Table

| Node Name        | Node Type                | Functional Role                       | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                                          |
|------------------|--------------------------|------------------------------------|---------------------|---------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger | Schedule Trigger          | Triggers workflow every hour       | None                | Config              | ## Slack - Errors last 60 minutes This workflow's objective is to monitor for errors and notify via Slack which workflows had failed executions in the last hour, providing a simple table with the ID, name, and error count. Linkedin: https://www.linkedin.com/in/matheus-pedrosa-custodio/ |
| Config           | Set                      | Sets URL base prefix for workflows | Schedule Trigger    | GetWorkflows        |                                                                                                                      |
| GetWorkflows     | n8n API                  | Retrieves active workflows          | Config              | Loop                |                                                                                                                      |
| Loop             | SplitInBatches           | Iterates workflows one by one       | GetWorkflows, FilterLastHour | MakeMessage, n8n    |                                                                                                                      |
| n8n              | n8n API                  | Queries latest failed execution per workflow | Loop             | Aggregate            |                                                                                                                      |
| Aggregate        | Aggregate                | Aggregates all error execution data | n8n                 | FilterLastHour       |                                                                                                                      |
| FilterLastHour   | Code                     | Filters executions from last hour   | Aggregate            | Loop                 |                                                                                                                      |
| MakeMessage      | Code                     | Creates Slack message blocks        | Loop                 | Slack                |                                                                                                                      |
| Slack            | Slack                    | Sends message to Slack user         | MakeMessage          | None                 |                                                                                                                      |
| Sticky Note      | Sticky Note              | Documentation note                  | None                 | None                 | ## Slack - Errors last 60 minutes This workflow's objective is to monitor for errors and notify via Slack which workflows had failed executions in the last hour, providing a simple table with the ID, name, and error count. Linkedin: https://www.linkedin.com/in/matheus-pedrosa-custodio/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Set interval to every 1 hour.
   - Position: Left-top area.

2. **Add Config Node**
   - Type: Set
   - Mode: Raw
   - JSON Output: `{ "urlBase": "https://n8n.com/workflow/" }`
   - Connect Schedule Trigger → Config.

3. **Add GetWorkflows Node**
   - Type: n8n API
   - Set filter: Active workflows only (`activeWorkflows: true`).
   - Credentials: Select or create n8n API credentials.
   - Connect Config → GetWorkflows.

4. **Add Loop Node**
   - Type: SplitInBatches
   - Default batch size (process one workflow at a time).
   - Connect GetWorkflows → Loop.

5. **Add n8n Node (to query failed executions)**
   - Type: n8n API
   - Resource: Execution
   - Filters:
     - Status: "error"
     - Workflow ID: Expression referencing current item workflow id (`{{$json.id}}`).
   - Limit results to 1.
   - Credentials: Use same n8n API credentials.
   - Connect Loop → n8n.

6. **Add Aggregate Node**
   - Type: Aggregate
   - Operation: Aggregate all items.
   - Connect n8n → Aggregate.

7. **Add FilterLastHour Node**
   - Type: Code node (JavaScript)
   - Code:
     ```javascript
     const list = $("Aggregate").last().json.data;
     const lastHour = new Date();
     lastHour.setHours(lastHour.getHours() - 1);
     const executionsLastHour = list.filter(execution => {
       const dataStoppedAt = new Date(execution.stoppedAt);
       return dataStoppedAt > lastHour;
     });
     return {
       workflowId: $("Loop").last().json.id,
       workflowName: $("Loop").last().json.name,
       qtdErrors: executionsLastHour.length,
       url: $("Config").last().json.urlBase + $("Loop").last().json.id
     };
     ```
   - Connect Aggregate → FilterLastHour.

8. **Connect FilterLastHour Back to Loop**
   - This allows the filtered error summary per workflow to flow into the message creation step.

9. **Add MakeMessage Node**
   - Type: Code node (JavaScript)
   - Code:
     ```javascript
     const list = $input.all().map(item => item.json);
     let blocks = [];
     for (const item of list) {
       if(item.qtdErrors > 0){
         blocks.push({
           "type": "section",
           "text": {
             "type": "mrkdwn",
             "text": `:aviso: *${item.workflowName}*\n *${item.qtdErrors}* errors found in the last run`
           },
           "accessory": {
             "type": "button",
             "text": {
               "type": "plain_text",
               "text": "Vizualize",
               "emoji": true
             },
             "value": "click_me_123",
             "url": `${item.url}`,
             "action_id": "button-action"
           }
         });
       }
     }
     if(blocks.length > 0){
       return { blocks: blocks };
     } else {
       return null;
     }
     ```
   - Connect Loop → MakeMessage.

10. **Add Slack Node**
    - Type: Slack
    - Authentication: OAuth2 credentials configured for Slack bot.
    - Message Type: Block
    - User: Set to Slack User ID (`U05FMSA2610`).
    - Blocks UI: Set expression to use output from MakeMessage node.
    - Connect MakeMessage → Slack.

11. **Add Sticky Note (optional)**
    - Add documentation note describing workflow purpose and author LinkedIn.
    - Place visibly near Schedule Trigger.

12. **Configure Credentials**
    - n8n API credentials must have access to workflow and execution data.
    - Slack OAuth2 credentials must have permission to send messages to the selected user.

13. **Set Workflow Settings**
    - Timezone: America/Sao_Paulo.
    - Execution timeout: Disabled (-1).
    - Execution order: Sequential (v1).
    - Save manual executions enabled.

14. **Activate Workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Workflow objective: Monitor n8n workflows for errors in last 60 minutes and notify via Slack with workflow ID, name, and error count.                                        | Sticky Note content in workflow.                                                                                          |
| LinkedIn profile of author for contact or professional reference: https://www.linkedin.com/in/matheus-pedrosa-custodio/                                                       | Sticky Note content in workflow.                                                                                          |
| Slack message uses Block Kit JSON format for rich message formatting with buttons linking to workflow URLs.                                                                    | Slack API documentation: https://api.slack.com/block-kit                                                                 |
| The workflow requires n8n API and Slack OAuth2 credentials to operate correctly.                                                                                                | n8n credentials setup docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n/                                      |
| Time filtering uses local timezone (America/Sao_Paulo); ensure system time matches or adjust accordingly.                                                                      | N8N workflow settings timezone.                                                                                           |

---

This documentation provides a thorough and structured understanding of the "Automated Hourly n8n Error Monitoring with Slack Notifications" workflow, enabling developers and AI systems to analyze, reproduce, and maintain it effectively.