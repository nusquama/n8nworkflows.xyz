Access execution data from an error workflow

https://n8nworkflows.xyz/workflows/access-execution-data-from-an-error-workflow-2349


# Access execution data from an error workflow

### 1. Workflow Overview

This workflow demonstrates how to access and utilize the execution data from a failed workflow run specifically within an error handling context in n8n. It targets use cases where error workflows need to branch or decide on remediation steps based on the data that was being processed when the failure occurred — for example, retrieving the payload from a webhook node that triggered the failed workflow.

The workflow’s logical structure is divided into three main blocks:

- **1.1 Error Trigger Reception:** Captures the error event from any workflow execution.
- **1.2 Execution Data Retrieval:** Uses the native n8n API node to fetch detailed data from the failed execution, including node run data.
- **1.3 Webhook Data Extraction:** Runs a JavaScript code node to parse the retrieved execution data, identify webhook nodes, and extract the payload of the webhook that was active during the failure.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Trigger Reception

- **Overview:**  
  This block listens for error events triggered by failed executions in any workflow. It acts as the entry point and initiates the error handling process by passing error metadata downstream.

- **Nodes Involved:**  
  - `Error Trigger`

- **Node Details:**  
  - **Node Name:** Error Trigger  
  - **Type:** `n8n-nodes-base.errorTrigger`  
  - **Technical Role:** Listens for all workflow execution errors globally. Triggers on any error event.  
  - **Configuration Choices:** No parameters configured; uses default to catch all errors.  
  - **Key Expressions/Variables:** Emits data including `execution.id` of the failed execution.  
  - **Input/Output Connections:** No input; outputs to `Get execution data`.  
  - **Version-Specific Requirements:** Available in n8n versions supporting Error Trigger node (v0.140+).  
  - **Potential Failures:** None internal; depends on n8n error event firing correctly.  
  - **Sub-workflow:** None.

#### 1.2 Execution Data Retrieval

- **Overview:**  
  Retrieves the full execution data of the failed workflow run using the n8n internal API node, enabling inspection of all node outputs at the time of failure.

- **Nodes Involved:**  
  - `Get execution data`

- **Node Details:**  
  - **Node Name:** Get execution data  
  - **Type:** `n8n-nodes-base.n8n` (n8n API node)  
  - **Technical Role:** Fetches execution details by its ID from n8n’s internal API.  
  - **Configuration Choices:**  
    - Operation: `get`  
    - Resource: `execution`  
    - Execution ID: dynamically set via expression `={{ $json.execution.id }}` from error trigger output.  
    - Options: `activeWorkflows` set to true (optional, to include active workflows in response).  
  - **Key Expressions/Variables:** Uses `$json.execution.id` from upstream Error Trigger node.  
  - **Input/Output Connections:** Input from `Error Trigger`; output to `Extract webhook data`.  
  - **Credentials:** Uses `n8nApi` credential configured with n8n internal API access.  
  - **Version-Specific Requirements:** Requires n8n API node and an API credential configured for the instance.  
  - **Potential Failures:**  
    - Auth errors if credentials are missing or invalid.  
    - Failure if execution ID is not found or execution data is purged.  
    - Timeout or API connectivity issues.  
  - **Sub-workflow:** None.

#### 1.3 Webhook Data Extraction

- **Overview:**  
  Processes the retrieved execution data to find nodes of type webhook and extract their payload data from the failed execution, enabling downstream conditional logic based on the original input.

- **Nodes Involved:**  
  - `Extract webhook data`

- **Node Details:**  
  - **Node Name:** Extract webhook data  
  - **Type:** `n8n-nodes-base.code` (JavaScript code node)  
  - **Technical Role:** Runs custom JavaScript to parse JSON data from the upstream node and extract webhook node names and their corresponding payloads.  
  - **Configuration Choices:**  
    - Mode: `runOnceForEachItem` (executes once per input item)  
    - Custom Code:  
      ```javascript
      const webhook_node_names = $json.workflowData.nodes.filter(x => x.type == 'n8n-nodes-base.webhook').map(x => x.name)

      const webhook_data_array = webhook_node_names.map(n => $json.data.resultData.runData[n] ? $json.data.resultData.runData[n][0].data.main[0][0].json : null).filter(x => x != null)

      let webhook_data = null;
      if (webhook_data_array.length > 0) {
        webhook_data = webhook_data_array[0]
      }

      return {
        'webhook_node_names': webhook_node_names,
        'webook_node_payload': webhook_data
      }
      ```  
    - This code:  
      - Identifies all webhook nodes in the workflow metadata  
      - Checks the run data of each webhook node in the execution's run data to get the payload  
      - Returns the first webhook payload found (if any) and the list of webhook node names.  
  - **Key Expressions/Variables:** Accesses `$json.workflowData.nodes`, `$json.data.resultData.runData`, and returns a structured object with webhook info.  
  - **Input/Output Connections:** Input from `Get execution data`; no downstream nodes in this workflow (can be extended).  
  - **Version-Specific Requirements:** Code node with JavaScript support (standard in n8n).  
  - **Potential Failures:**  
    - If execution data format changes, expressions may fail.  
    - If no webhook nodes exist or run data is empty, returns null payload.  
    - Potential typos in returned property name (`webook_node_payload` instead of `webhook_node_payload`) — might cause confusion.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type                    | Functional Role                     | Input Node(s)   | Output Node(s)        | Sticky Note                                                                                              |
|---------------------|------------------------------|-----------------------------------|-----------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Error Trigger       | errorTrigger                 | Capture error events from workflows | None            | Get execution data    |                                                                                                        |
| Get execution data  | n8n API                      | Retrieve full execution data for failed run | Error Trigger  | Extract webhook data  | Requires `n8nApi` credential with access to internal API                                               |
| Extract webhook data| Code (JavaScript)            | Parse execution data to extract webhook node payloads | Get execution data | None                  | This template shows how to retrieve webhook data from an error workflow execution                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Error Trigger Node**  
   - Add node of type `Error Trigger`.  
   - No parameters required. This node listens globally to all workflow errors.

2. **Create an n8n API Node to Get Execution Data**  
   - Add node of type `n8n` (API node).  
   - Set Resource to `execution`.  
   - Set Operation to `get`.  
   - Set `executionId` parameter with the expression: `={{ $json.execution.id }}` (this pulls the failed execution id from the Error Trigger node).  
   - Under Options, enable `activeWorkflows` (optional).  
   - Configure credentials: Create or select `n8nApi` credentials linked to your n8n instance API with sufficient permissions.  
   - Connect the output of `Error Trigger` to this node.

3. **Create a Code Node to Extract Webhook Data**  
   - Add a `Code` node.  
   - Set mode to `Run Once For Each Item`.  
   - Paste the following JavaScript code:  
     ```javascript
     const webhook_node_names = $json.workflowData.nodes.filter(x => x.type == 'n8n-nodes-base.webhook').map(x => x.name);

     const webhook_data_array = webhook_node_names.map(n => $json.data.resultData.runData[n] ? $json.data.resultData.runData[n][0].data.main[0][0].json : null).filter(x => x != null);

     let webhook_data = null;
     if (webhook_data_array.length > 0) {
       webhook_data = webhook_data_array[0];
     }

     return {
       'webhook_node_names': webhook_node_names,
       'webook_node_payload': webhook_data
     };
     ```  
   - Connect the output of the n8n API node (`Get execution data`) to this node.

4. **Link the Nodes**  
   - Connect `Error Trigger` → `Get execution data` → `Extract webhook data`.

5. **Save and Activate the Workflow**  
   - Save your workflow.  
   - Activate it to listen for errors in other workflows.

6. **Testing**  
   - Trigger an error in a workflow that includes a webhook node.  
   - The error workflow will start, fetch execution data, and extract the webhook payload for your error handling logic.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow template illustrates advanced error handling by accessing failed execution data | Useful for dynamic error workflows that need context-aware remediation                                      |
| OpenAI credential is not required here but n8nApi credential is mandatory for internal API calls | Configure n8nApi credential with your instance’s API key and URL                                            |
| For more details on Error Trigger node, see: https://docs.n8n.io/nodes/n8n-nodes-base.errortrigger/ | Official n8n documentation                                                                                  |
| The template focuses on webhook node data extraction but can be adapted for other node types  | Modify code node logic to parse other node types’ data if needed                                           |

---

This documentation enables developers and automation agents to understand, reproduce, and extend the error workflow for context-aware error handling based on execution data in n8n.