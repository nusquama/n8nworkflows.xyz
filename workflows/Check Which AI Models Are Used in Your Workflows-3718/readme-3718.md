Check Which AI Models Are Used in Your Workflows

https://n8nworkflows.xyz/workflows/check-which-ai-models-are-used-in-your-workflows-3718


# Check Which AI Models Are Used in Your Workflows

### 1. Workflow Overview

This workflow is designed to audit and report which AI models are used across all workflows in an n8n instance. It targets users who want to inventory AI model usage for governance, optimization, or documentation purposes.

The workflow logically divides into these blocks:

- **1.1 Trigger and Initialization:** Manual trigger to start the process and clear old data in the Google Sheet.
- **1.2 Workflow Retrieval and Filtering:** Fetch all workflows via n8n API and filter those containing nodes with AI model identifiers (`modelId`).
- **1.3 Node Extraction and Filtering:** For each filtered workflow, extract nodes and filter nodes that actually specify a `modelId` or `model`.
- **1.4 Data Transformation and Output:** Format extracted data fields and append the results into a connected Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:** Starts the workflow manually and clears previous data from the Google Sheet to prepare for fresh results.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Google Sheets-Clear Sheet Data

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to "n8n-get all workflow" node.  
    - Edge cases: None typical; user must manually trigger.  

  - **Google Sheets-Clear Sheet Data**  
    - Type: Google Sheets node  
    - Role: Clears existing data in the target sheet to avoid appending duplicates.  
    - Configuration:  
      - Operation: Clear  
      - Sheet Name: Uses sheet ID `gid=0` from the configured Google Sheet document.  
      - Document ID: The Google Sheet ID where results are stored.  
      - Credentials: OAuth2 Google Sheets account connected.  
    - Inputs: Triggered after filtering workflows with modelId.  
    - Outputs: None (executeOnce: true)  
    - Edge cases:  
      - Google API auth failure  
      - Sheet ID or document ID misconfiguration  
      - Network timeouts  

#### 2.2 Workflow Retrieval and Filtering

- **Overview:** Retrieves all workflows from the n8n instance and filters to keep only those containing nodes with a `modelId` property.
- **Nodes Involved:**  
  - n8n-get all workflow  
  - Filter-get workflow contain modelid

- **Node Details:**

  - **n8n-get all workflow**  
    - Type: n8n API node  
    - Role: Calls the n8n API to list all workflows.  
    - Configuration: No filters applied; fetches all workflows.  
    - Credentials: n8n API credentials connected.  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Passes workflows to filter node and triggers Google Sheets clear node.  
    - Edge cases:  
      - API authentication failure  
      - Large number of workflows causing timeout or slow response  

  - **Filter-get workflow contain modelid**  
    - Type: Filter node  
    - Role: Filters workflows whose nodes contain the string "modelId" in their JSON representation and excludes the current workflow itself.  
    - Configuration:  
      - Condition: Checks if the stringified nodes JSON contains "modelId"  
      - Excludes current workflow by comparing workflow IDs  
    - Inputs: From n8n-get all workflow  
    - Outputs: Passes filtered workflows to "Loop Over Items" node  
    - Edge cases:  
      - Workflows without nodes property (unlikely)  
      - False positives if "modelId" appears in unrelated text (unlikely)  

#### 2.3 Node Extraction and Filtering

- **Overview:** Splits each filtered workflow into individual nodes, then filters nodes that actually have a `modelId` or `model` parameter defined.
- **Nodes Involved:**  
  - Loop Over Items  
  - Split Out-nodes  
  - Filter-node contain modelId

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes workflows one by one to handle large data sets efficiently.  
    - Configuration: Default batch options (no batch size specified).  
    - Inputs: From Filter-get workflow contain modelid  
    - Outputs: Splits workflows to "Split Out-nodes" node  
    - Edge cases:  
      - Large number of workflows may slow processing  
      - Batch size defaults may affect performance  

  - **Split Out-nodes**  
    - Type: SplitOut  
    - Role: Splits each workflow’s nodes array into individual node items for filtering.  
    - Configuration: Field to split out is "nodes".  
    - Inputs: From Loop Over Items (second output)  
    - Outputs: Passes individual nodes to Filter-node contain modelId  
    - Edge cases:  
      - Workflows with empty or missing nodes array  

  - **Filter-node contain modelId**  
    - Type: Filter node  
    - Role: Filters nodes that have either a `modelId.value` or `model` parameter defined (checking existence).  
    - Configuration:  
      - Condition: Checks if `parameters.modelId.value` or `parameters.model` exists and is non-empty.  
    - Inputs: From Split Out-nodes  
    - Outputs: Passes filtered nodes to Edit Fields-set_model_data  
    - Edge cases:  
      - Nodes with model info stored differently may be missed  
      - Nodes with empty or null model fields  

#### 2.4 Data Transformation and Output

- **Overview:** Formats the extracted node and workflow data into a structured form and appends it to the Google Sheet.
- **Nodes Involved:**  
  - Edit Fields-set_model_data  
  - Google Sheets-Save node and workflow data

- **Node Details:**

  - **Edit Fields-set_model_data**  
    - Type: Set node  
    - Role: Creates a structured JSON object with relevant fields for output.  
    - Configuration:  
      - Assigns:  
        - `node_name`: node’s name  
        - `model`: tries to extract model from multiple possible fields (`parameters.model.value`, `parameters.model`, or `parameters.modelId.cachedResultName`)  
        - `workflow_name`: current workflow name from Loop Over Items  
        - `workflow_id`: current workflow ID from Loop Over Items  
        - `workflow_url`: constructed URL using a placeholder domain `Your-n8n-domain` concatenated with workflow and node IDs (user must replace domain)  
    - Inputs: From Filter-node contain modelId  
    - Outputs: Passes formatted data to Google Sheets append node  
    - Edge cases:  
      - Missing or unexpected model fields may result in empty or incorrect model values  
      - User must replace domain placeholder for URLs to be valid  

  - **Google Sheets-Save node and workflow data**  
    - Type: Google Sheets node  
    - Role: Appends the formatted data rows into the Google Sheet.  
    - Configuration:  
      - Operation: Append  
      - Sheet Name and Document ID: same as clear node, pointing to the configured Google Sheet  
      - Columns: Maps input fields to columns `node_name`, `modelId_value`, `modelId_name`, `workflow_name`, `workflow_id`, `workflow_url`  
      - Credentials: OAuth2 Google Sheets account connected  
    - Inputs: From Edit Fields-set_model_data  
    - Outputs: Loops back to Loop Over Items to continue processing  
    - Edge cases:  
      - Google API errors or quota limits  
      - Data mapping errors if input fields are missing or malformed  

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                      | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                      |
|----------------------------------|---------------------|-----------------------------------------------------|-------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger      | Starts the workflow manually                         | None                          | n8n-get all workflow             |                                                                                                |
| n8n-get all workflow              | n8n API             | Fetches all workflows from n8n instance             | When clicking ‘Test workflow’ | Filter-get workflow contain modelid, Google Sheets-Clear Sheet Data |                                                                                                |
| Filter-get workflow contain modelid | Filter              | Filters workflows containing nodes with `modelId`  | n8n-get all workflow          | Loop Over Items                  |                                                                                                |
| Google Sheets-Clear Sheet Data   | Google Sheets       | Clears old data from the Google Sheet                | Filter-get workflow contain modelid | None (executeOnce)               |                                                                                                |
| Loop Over Items                  | SplitInBatches      | Processes workflows one by one                        | Filter-get workflow contain modelid | Split Out-nodes                 |                                                                                                |
| Split Out-nodes                 | SplitOut            | Splits workflows into individual nodes               | Loop Over Items               | Filter-node contain modelId      |                                                                                                |
| Filter-node contain modelId      | Filter              | Filters nodes that contain `modelId` or `model`      | Split Out-nodes              | Edit Fields-set_model_data       |                                                                                                |
| Edit Fields-set_model_data       | Set                 | Formats node and workflow data for output            | Filter-node contain modelId   | Google Sheets-Save node and workflow data |                                                                                                |
| Google Sheets-Save node and workflow data | Google Sheets       | Appends extracted data rows into Google Sheet        | Edit Fields-set_model_data    | Loop Over Items                  |                                                                                                |
| Sticky Note                     | Sticky Note         | Reminder to change domain placeholder                 | None                         | None                           | ## Change to your n8n domain here                                                              |
| Sticky Note1                    | Sticky Note         | Warning about performance with >100 workflows         | None                         | None                           | ## Be careful if you have more than 100 workflows. It might have performance issue.             |
| Sticky Note4                    | Sticky Note         | Author and contact information                         | None                         | None                           | ## Created by darrell_tw_ \n\nAn engineer now focus on AI and Automation\n\n### contact me with following:\n[X](https://x.com/darrell_tw_)\n[Threads](https://www.threads.net/@darrell_tw_)\n[Instagram](https://www.instagram.com/darrell_tw_/)\n[Website](https://www.darrelltw.com/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create n8n API node to fetch workflows**  
   - Name: `n8n-get all workflow`  
   - Type: n8n API  
   - Operation: List workflows (default)  
   - Credentials: Connect your n8n API credentials.  
   - Connect output from Manual Trigger node.

3. **Create Filter node to keep workflows with modelId**  
   - Name: `Filter-get workflow contain modelid`  
   - Type: Filter  
   - Condition:  
     - Check if stringified JSON of workflow nodes contains `"modelId"`  
     - AND workflow ID is not equal to current workflow ID (to exclude self)  
   - Connect input from n8n-get all workflow node.

4. **Create Google Sheets node to clear old data**  
   - Name: `Google Sheets-Clear Sheet Data`  
   - Type: Google Sheets  
   - Operation: Clear  
   - Document ID: Use your Google Sheet ID (from the template or your own sheet)  
   - Sheet Name: Use `gid=0` or your target sheet tab ID  
   - Credentials: Connect your Google Sheets OAuth2 account  
   - Connect input from Filter-get workflow contain modelid node (parallel to Loop Over Items).

5. **Create SplitInBatches node to loop over workflows**  
   - Name: `Loop Over Items`  
   - Type: SplitInBatches  
   - Default batch size is fine unless you want to optimize.  
   - Connect input from Filter-get workflow contain modelid node.

6. **Create SplitOut node to split workflows into nodes**  
   - Name: `Split Out-nodes`  
   - Type: SplitOut  
   - Field to split out: `nodes`  
   - Connect input from second output of Loop Over Items node.

7. **Create Filter node to keep nodes with modelId or model**  
   - Name: `Filter-node contain modelId`  
   - Type: Filter  
   - Condition:  
     - Check if `parameters.modelId.value` exists and is non-empty  
     - OR `parameters.model` exists and is non-empty  
   - Connect input from Split Out-nodes node.

8. **Create Set node to format output data**  
   - Name: `Edit Fields-set_model_data`  
   - Type: Set  
   - Assign fields:  
     - `node_name`: `{{$json.name}}`  
     - `model`: `{{$json.parameters.model.value || $json.parameters.model || $json.parameters.modelId.cachedResultName}}`  
     - `workflow_name`: `{{$node["Loop Over Items"].item.json.name}}`  
     - `workflow_id`: `{{$node["Loop Over Items"].item.json.id}}`  
     - `workflow_url`: Replace `Your-n8n-domain` with your actual domain in:  
       `={Your-n8n-domain}/workflow/{{$node["Loop Over Items"].item.json.id}}/{{$json.id}}`  
   - Connect input from Filter-node contain modelId node.

9. **Create Google Sheets node to append data**  
   - Name: `Google Sheets-Save node and workflow data`  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID and Sheet Name: Same as clear node  
   - Columns mapping: Map input fields to columns:  
     - `node_name`  
     - `modelId_value` (mapped from `model`)  
     - `modelId_name` (optional, can be left empty or same as model)  
     - `workflow_name`  
     - `workflow_id`  
     - `workflow_url`  
   - Credentials: Connect your Google Sheets OAuth2 account  
   - Connect input from Edit Fields-set_model_data node.

10. **Connect output of Google Sheets append node back to Loop Over Items**  
    - This creates a loop to process all workflows.

11. **Add Sticky Notes for guidance**  
    - Add a sticky note near the domain placeholder reminding to replace `"Your-n8n-domain"` with your actual domain URL.  
    - Add a sticky note warning about performance issues if you have more than 100 workflows.  
    - Add a sticky note with author contact info if desired.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Use this [Google Sheet template](https://docs.google.com/spreadsheets/d/1vl_Xj9QLGE04XMimmJzj1V0vJgP3j18duhsIziVNx_A/copy) to create your results sheet. | Google Sheets template for storing AI model usage results.                                                          |
| Be cautious if you have more than 100 workflows; performance may degrade due to API and processing overhead.          | Sticky note warning inside the workflow.                                                                             |
| Replace `"Your-n8n-domain"` in the URL construction with your actual n8n instance domain to generate valid URLs.      | Sticky note inside the workflow near the URL field.                                                                  |
| Created by darrell_tw_, an engineer focused on AI and Automation. Contact via [X](https://x.com/darrell_tw_), [Threads](https://www.threads.net/@darrell_tw_), [Instagram](https://www.instagram.com/darrell_tw_/), [Website](https://www.darrelltw.com/) | Author and contact information sticky note.                                                                          |
| The workflow clears old data before writing new results to avoid duplication.                                         | Workflow design note.                                                                                                |
| Ensure your n8n instance allows API access and your credentials have sufficient permissions for workflow listing.     | Setup prerequisite.                                                                                                  |
| The workflow now correctly detects AI models in the "tool" originally missed (update note).                          | Workflow update note.                                                                                                |