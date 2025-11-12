Backup n8n Workflows with Versioning and Notion Tracking

https://n8nworkflows.xyz/workflows/backup-n8n-workflows-with-versioning-and-notion-tracking-6947


# Backup n8n Workflows with Versioning and Notion Tracking

### 1. Workflow Overview

This workflow automates the backup of n8n workflows from a source n8n instance to a destination instance with versioning and tracking via Notion. It is designed to:

- Retrieve all workflows from a source n8n instance.
- Copy workflows to a destination n8n instance.
- Prefix workflow names on the destination with the current date in `YYYY-MM-DD_` format to maintain multiple versions.
- Maintain a rolling retention policy by deleting workflows older than a configurable number of days.
- Record the date of backup execution and the number of workflows processed in a Notion database for tracking.

The workflow is logically organized into the following blocks:

- **1.1 Trigger and Date Calculation:** Initiates the workflow manually and calculates relevant date prefixes for naming and retention.
- **1.2 Source Workflow Retrieval:** Fetches workflows from the source instance.
- **1.3 Destination Workflow Management:** Retrieves workflows from the destination, filters old versions for deletion, and deletes them.
- **1.4 Workflow Copying and Versioning:** Loops over source workflows to create dated copies on the destination.
- **1.5 Notion Tracking:** Updates a Notion database with backup metadata.
- **1.6 Control and Conditional Logic:** Checks if backup was already performed for the day and branches accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Date Calculation

**Overview:**  
This block triggers the workflow manually and calculates the current date, yesterday's date, and corresponding formatted prefixes used for naming workflows and retention policy.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- today  
- yesterday  
- yesterday_prefix  
- today_prefix  
- Sticky Note6 (Substract From Date explanation)  
- Sticky Note7 (Date format explanation)  
- Sticky Note9 (Prefix explanation)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Inputs: None  
  - Outputs: Triggers `today` node.  
  - Potential Failures: None, manual trigger.

- **today**  
  - Type: Date & Time  
  - Role: Generates the current date/time for the run.  
  - Configuration: Default current date/time.  
  - Outputs: Current date/time passed to `today_prefix` and `yesterday` nodes.

- **yesterday**  
  - Type: Date & Time  
  - Role: Calculates the date two days before the current date (for rolling retention).  
  - Configuration: Subtracts 2 days from `today` date.  
  - Outputs: New date forwarded to `yesterday_prefix` and `GET - Workflows` node.  
  - Edge Cases: Timezone differences may affect date calculations.

- **yesterday_prefix**  
  - Type: Date & Time  
  - Role: Formats the `yesterday` date to `yyyy-MM-dd_` string for workflow filtering/deletion.  
  - Configuration: Custom format: `yyyy-MM-dd_`.  
  - Outputs: Used by `GET - Destination Workflows` to identify workflows to delete.

- **today_prefix**  
  - Type: Date & Time  
  - Role: Formats the `today` date to `yyyy-MM-dd_` string for naming new workflows and Notion tracking.  
  - Configuration: Custom format: `yyyy-MM-dd_`.  
  - Outputs: Passed to `Notion1` (Notion fetch) and `CREATE - Workflow` for naming.

- **Sticky Notes (6,7,9)**  
  - Role: Provide instructions and explanations on date subtraction, format, and usage of prefix for naming.

---

#### 2.2 Source Workflow Retrieval

**Overview:**  
Fetches all workflows from the source n8n instance that will be backed up.

**Nodes Involved:**  
- GET - Workflows  
- Limit (disabled)  
- Split Out Workflows

**Node Details:**

- **GET - Workflows**  
  - Type: n8n API (n8n node)  
  - Role: Retrieves workflows from the source n8n instance.  
  - Configuration: No filters or request options, retrieves all workflows.  
  - Credentials: Source n8n API credentials.  
  - Outputs: Workflow list sent to `Limit` node.

- **Limit** (disabled)  
  - Type: Limit  
  - Role: Intended to limit number of workflows processed (disabled for full run).  
  - Configuration: Default limit off.  
  - Outputs: Passes workflows to `Split Out Workflows`.

- **Split Out Workflows**  
  - Type: Split Out  
  - Role: Splits workflow list into individual items for looping.  
  - Configuration: Splits on `id` field including all other fields.  
  - Outputs: Individual workflows forwarded to `Loop Over Items`.

---

#### 2.3 Destination Workflow Management

**Overview:**  
Retrieves existing workflows from the destination instance to manage version retention by deleting workflows older than the retention period.

**Nodes Involved:**  
- GET - Destination Workflows  
- Split Out Workflows1  
- Filter  
- n8n_destination (Delete node)  
- Sticky Note4  
- Sticky Note5

**Node Details:**

- **GET - Destination Workflows**  
  - Type: n8n API (n8n node)  
  - Role: Fetches workflows from the destination instance.  
  - Configuration: Limit 200, batching enabled.  
  - Credentials: Destination n8n API credentials.

- **Split Out Workflows1**  
  - Type: Split Out  
  - Role: Splits destination workflows into individual items.  
  - Configuration: Splits on `id`.

- **Filter**  
  - Type: Filter  
  - Role: Selects workflows whose names start with the date prefix corresponding to the day before yesterday (i.e., workflows to delete).  
  - Configuration: Compares first 11 characters of workflow name to `yesterday_prefix` date.  
  - Outputs: Workflows to delete passed to `n8n_destination`.

- **n8n_destination**  
  - Type: n8n API (n8n node)  
  - Role: Deletes filtered workflows on the destination instance.  
  - Configuration: Delete operation specifying workflow ID from filter.  
  - Credentials: Destination n8n API credentials.

- **Sticky Note4 & Sticky Note5**  
  - Role: Explain the deletion of old workflows and selection of all workflows on destination.

---

#### 2.4 Workflow Copying and Versioning

**Overview:**  
Loops through each source workflow and creates a copy on the destination instance, prefixing the workflow name with the current date for versioning. Includes a wait to avoid API rate limits.

**Nodes Involved:**  
- Loop Over Items  
- CREATE - Workflow  
- Wait  
- Sticky Note2 (Destination Save Today)  
- Sticky Note8 (Try with 1 first)

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes workflows one-by-one or in batches.  
  - Configuration: Default batch size (not explicitly set).  
  - Inputs: Individual source workflows.  
  - Outputs: Passes each workflow to `CREATE - Workflow` and `Notion`.

- **CREATE - Workflow**  
  - Type: n8n API (n8n node)  
  - Role: Creates a new workflow on the destination instance.  
  - Configuration:  
    - Operation: Create  
    - Workflow object: JSON structured with name prefixed by `today_prefix` + original name, nodes, and connections copied from source.  
  - Credentials: Destination n8n API credentials.  
  - Inputs: Individual source workflow data with prefix injected.  
  - Outputs: Passes to `Wait` node.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow execution for 1 second to prevent hitting API limits.  
  - Configuration: Amount 1 (second).  
  - Inputs: From `CREATE - Workflow`.  
  - Outputs: Passes to `Loop Over Items` for next item.

- **Sticky Note2**  
  - Role: Explains naming convention for destination workflows, prefixing name with `YYYY-MM-DD_`.

- **Sticky Note8**  
  - Role: Suggests testing with batch size 1 first.

---

#### 2.5 Notion Tracking

**Overview:**  
Updates a Notion database page with metadata about the backup, including the current date prefix and the number of workflows processed.

**Nodes Involved:**  
- Notion1 (Get)  
- If  
- Notion (Update)  
- backup_already_done (NoOp)  
- Sticky Note (in main block explaining overall workflow)

**Node Details:**

- **Notion1**  
  - Type: Notion API  
  - Role: Retrieves the Notion database page to check existing backup info.  
  - Configuration: Page ID set to specific database page for tracking.  
  - Credentials: Notion API credentials.  
  - Outputs: Data passed to `If`.

- **If**  
  - Type: If  
  - Role: Checks if backup for the current date prefix already exists (by comparing property value).  
  - Configuration: Compares Notion page property `property_value` to `today_prefix`.  
  - Outputs:  
    - If true: Sends workflow to `backup_already_done` (NoOp).  
    - If false: Proceeds with backup workflow.

- **Notion**  
  - Type: Notion API  
  - Role: Updates the Notion database page with current date prefix and the number of workflows backed up (length of items).  
  - Configuration: Updates "Value" with `today_prefix` and "Comment" with count as string.  
  - Executes once per workflow run.  
  - Credentials: Notion API credentials.

- **backup_already_done**  
  - Type: NoOp (No operation)  
  - Role: Terminates workflow branch if backup already done to avoid duplication.

---

#### 2.6 Control and Conditional Logic

**Overview:**  
Coordinates the workflow execution flow based on whether backup was already performed for the day.

**Nodes Involved:**  
- If  
- backup_already_done  
- today_prefix (feeds into Notion check)  
- Notion1  

**Node Details:**

- **If** node controls branching:  
  - If backup exists for today, ends in `backup_already_done`.  
  - Else proceeds to main backup logic.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                          | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                      |
|--------------------------|---------------------|----------------------------------------|-------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Initiates the workflow manually        | None                          | today                           |                                                                                                |
| today                    | Date & Time         | Gets current date/time                   | When clicking ‘Test workflow’  | today_prefix, yesterday         |                                                                                                |
| yesterday                | Date & Time         | Calculates date 2 days before today     | today                         | yesterday_prefix, GET - Workflows| Sticky Note6: "Substract 2 (duration) for the day before yesterday..."                          |
| yesterday_prefix         | Date & Time         | Formats yesterday date as prefix        | yesterday                     | GET - Destination Workflows     | Sticky Note7: "Setup the custom format with yyyy-MM-dd_"                                       |
| today_prefix             | Date & Time         | Formats today's date as prefix           | today                         | Notion1, CREATE - Workflow      | Sticky Note9: "This is the prefix added to workflow name"                                      |
| GET - Workflows          | n8n API             | Retrieves workflows from source n8n     | yesterday_prefix              | Limit                          |                                                                                                |
| Limit (disabled)         | Limit               | Limits workflows processed (disabled)   | GET - Workflows               | Split Out Workflows             | Sticky Note8: "Try with 1 first"                                                               |
| Split Out Workflows      | Split Out           | Splits source workflows for looping     | Limit                        | Loop Over Items                 |                                                                                                |
| Loop Over Items          | Split In Batches    | Loops over source workflows              | Split Out Workflows           | Notion, CREATE - Workflow       |                                                                                                |
| CREATE - Workflow        | n8n API             | Creates workflow copy on destination    | Loop Over Items              | Wait                          | Sticky Note2: "Destination Save Today: workflow name becomes YYYY-MM-DD_workflowname"          |
| Wait                     | Wait                | Delays to avoid API rate limits          | CREATE - Workflow             | Loop Over Items                |                                                                                                |
| GET - Destination Workflows | n8n API           | Retrieves destination workflows          | yesterday_prefix             | Split Out Workflows1           | Sticky Note5: "Destination Select All"                                                        |
| Split Out Workflows1     | Split Out           | Splits destination workflows             | GET - Destination Workflows   | Filter                        |                                                                                                |
| Filter                   | Filter              | Filters workflows to delete by prefix    | Split Out Workflows1          | n8n_destination               | Sticky Note4: "Destination Delete the day before yesterday workflows starting with YYYY-MM-DD_"|
| n8n_destination          | n8n API             | Deletes old workflows on destination     | Filter                       | None                         |                                                                                                |
| Notion1                  | Notion API          | Retrieves Notion tracking page           | today_prefix                 | If                           |                                                                                                |
| If                       | If                  | Checks if backup already done             | Notion1                      | backup_already_done, yesterday |                                                                                                |
| backup_already_done      | NoOp                | Ends workflow if backup already done      | If (true branch)             | None                         |                                                                                                |
| Notion                   | Notion API          | Updates Notion with backup info           | Loop Over Items (once)        | None                         |                                                                                                |
| Sticky Notes (various)   | Sticky Note         | Provides explanations and instructions   | None                        | None                         | See detailed notes in Block 2 sections                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add **Manual Trigger** node named "When clicking ‘Test workflow’".

2. **Add Date Nodes:**  
   - Add **Date & Time** node "today" connected to trigger; default current date/time.  
   - Add **Date & Time** node "yesterday" connected to "today": configure operation "Subtract From Date" to subtract 2 days from `today`.  
   - Add **Date & Time** node "yesterday_prefix" connected to "yesterday": format date as `yyyy-MM-dd_` (custom format).  
   - Add **Date & Time** node "today_prefix" connected to "today": format date as `yyyy-MM-dd_`.

3. **Retrieve Source Workflows:**  
   - Add **n8n API** node "GET - Workflows" connected to "yesterday_prefix".  
   - Configure with source n8n credentials.  
   - No filters; retrieves all workflows.

4. **Split Source Workflows:**  
   - Add **Limit** node (optional, can be disabled).  
   - Connect "GET - Workflows" to "Limit".  
   - Add **Split Out** node "Split Out Workflows" connected to "Limit".  
   - Configure to split on `id` including all other fields.

5. **Loop Source Workflows:**  
   - Add **Split In Batches** node "Loop Over Items" connected to "Split Out Workflows".  
   - Use default batch size or set to 1 for testing.

6. **Create Workflows on Destination:**  
   - Add **n8n API** node "CREATE - Workflow" connected to "Loop Over Items".  
   - Configure operation "create" with workflow object JSON:  
     ```
     {
       "name": "{{ $json.formattedDate + $json.name }}",
       "nodes": {{ JSON.stringify($json.nodes) }},
       "connections": {{ JSON.stringify($json.connections || {}) }}
     }
     ```  
   - Use destination n8n credentials.

7. **Add Wait Node:**  
   - Connect "CREATE - Workflow" to **Wait** node with 1 second delay.  
   - Connect "Wait" output back to "Loop Over Items" for next item.

8. **Retrieve Destination Workflows:**  
   - Add **n8n API** node "GET - Destination Workflows" connected to "yesterday_prefix".  
   - Configure to retrieve workflows from destination instance.

9. **Split Destination Workflows:**  
   - Add **Split Out** node "Split Out Workflows1" connected to "GET - Destination Workflows".  
   - Split by `id`.

10. **Filter Old Workflows:**  
    - Add **Filter** node connected to "Split Out Workflows1".  
    - Configure condition to match workflows where first 11 characters of name equal `yesterday_prefix` string.

11. **Delete Old Workflows:**  
    - Add **n8n API** node "n8n_destination" configured to delete workflows by ID from filter output.  
    - Connect "Filter" node output to "n8n_destination".

12. **Notion Tracking Setup:**  
    - Add **Notion** node "Notion1" connected to "today_prefix".  
    - Configure to get database page by ID, using Notion credentials.

13. **Check Backup Already Done:**  
    - Add **If** node connected to "Notion1".  
    - Condition: check if Notion `property_value` equals `today_prefix`.  
    - True branch: connect to **NoOp** node "backup_already_done" to end workflow.  
    - False branch: continue backup process.

14. **Update Notion Tracking:**  
    - Connect "Loop Over Items" to **Notion** node (update operation).  
    - Configure to update database page properties:  
      - `Value` (rich text) with `today_prefix`.  
      - `Comment` (rich text) with count of workflows processed (`$items().length.toString()`).

15. **Finalize Connections:**  
    - Ensure nodes are connected respecting the logical flow and error handling.

16. **Credentials:**  
    - Set up credentials for source and destination n8n API nodes (API keys).  
    - Set up Notion credentials with appropriate access to the target database page.

17. **Optional:**  
    - Add Sticky Notes with explanations as per original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Inspired by [Alex Kim's workflow](https://n8n.io/workflows/3048-clone-n8n-workflows-between-instances-using-n8n-api/), this workflow enhances versioning by prefixing workflow names with date and tracking backups in Notion.                                                                                                                                                              | Original inspiration link                                                                                 |
| The workflow was tested on n8n version 1.103.2 (Ubuntu). Ensure API permissions (read, create, delete workflows) are correctly configured on both source and destination n8n instances.                                                                                                                                                                                              | Version and permission requirements                                                                       |
| Notion database must have three fields: `sequence` (with value "prefix"), `Value` (rich text, stores date string), and `Comment` (rich text, stores number of workflows).                                                                                                                                                                                                             | Notion database setup instructions                                                                        |
| Rolling retention policy is configurable by adjusting the subtraction days in the "Subtract From Date" node (`yesterday` node). Default is 2 days to keep 2 days of backups online.                                                                                                                                                                                                 | Retention policy adjustment                                                                                 |
| For help or questions, reach out via n8n community forum: https://community.n8n.io/ or LinkedIn contact of the author.                                                                                                                                                                                                                                                                | Community support links                                                                                    |
| This workflow protects backups by avoiding redundant runs for the same day via the Notion check and conditional branching.                                                                                                                                                                                                                                                          | Workflow idempotency and prevention of duplicate backups                                                 |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.