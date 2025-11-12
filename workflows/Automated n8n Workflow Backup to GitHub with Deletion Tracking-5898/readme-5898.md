Automated n8n Workflow Backup to GitHub with Deletion Tracking

https://n8nworkflows.xyz/workflows/automated-n8n-workflow-backup-to-github-with-deletion-tracking-5898


# Automated n8n Workflow Backup to GitHub with Deletion Tracking

### 1. Workflow Overview

This n8n workflow automates the backup of all n8n instance workflows to a GitHub repository, including tracking and deleting workflows on GitHub that were removed from n8n. The workflow ensures that each n8n workflow is saved as a JSON file named after the workflow ID, maintains synchronization between the local n8n instance and the GitHub repo, and handles new, updated, and deleted workflows distinctly.

Logical blocks:

- **1.1 Trigger and Initialization:** Starts the process either manually or on a schedule; sets global configuration such as GitHub repo owner and name.
- **1.2 Fetch Current n8n Workflows:** Retrieves all workflows from the current n8n instance.
- **1.3 Fetch GitHub Stored Workflows:** Lists files in the backup GitHub repository and fetches individual workflow JSON files.
- **1.4 Compare Workflows:** Checks if workflows on GitHub are new, different, or the same compared to the current n8n workflows, using base64 decoding and JSON ordering.
- **1.5 Update GitHub Repository:** Depending on the comparison, creates new files, edits existing ones, or does nothing.
- **1.6 Deleted Workflow Detection and Removal:** Detects workflows deleted in the n8n instance and deletes corresponding files from GitHub.
- **1.7 Subworkflow Execution and Loop Management:** Implements batch processing and self-invocation to manage memory and processing large numbers of workflows efficiently.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:** This block starts the workflow, either manually or on a schedule, and sets global variables for GitHub repository owner and name.
- **Nodes Involved:**  
  - On clicking 'execute'  
  - Schedule Trigger  
  - Globals  

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow.  
    - Configuration: Default trigger, no parameters.  
    - Input/Output: Outputs to Globals node.  
    - Failure Modes: None expected; manual node.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow daily at 7 AM.  
    - Configuration: Set to trigger at hour 7 daily.  
    - Input/Output: Outputs to Globals node.  
    - Failure Modes: Network or scheduling misconfiguration.

  - **Globals**  
    - Type: Set  
    - Role: Sets global variables for GitHub repo owner and repo name.  
    - Configuration:  
      - repo.owner: "john-doe" (replace with actual GitHub username)  
      - repo.name: "n8n-backup" (replace with actual repo name)  
    - Input: From either manual or schedule trigger.  
    - Output: To "Get many workflows" node.  
    - Failure Modes: Misconfiguration can cause repository API calls to fail.

---

#### 2.2 Fetch Current n8n Workflows

- **Overview:** Retrieves the list of all workflows from the current n8n instance via the n8n API.
- **Nodes Involved:**  
  - Get many workflows  
  - Workflows  
  - n8n  

- **Node Details:**

  - **Get many workflows**  
    - Type: n8n API node  
    - Role: Fetches all workflows from the n8n instance.  
    - Configuration: No filters applied, fetch all workflows.  
    - Credentials: Uses "n8n account" API credentials.  
    - Input: From Globals node.  
    - Output: To Workflows and n8n nodes.  
    - Failure Modes: API connectivity issues, auth errors.

  - **Workflows**  
    - Type: Aggregate  
    - Role: Aggregates all workflow data into a single array for further processing.  
    - Configuration: Uses "aggregateAllItemData" method.  
    - Input: From Get many workflows node.  
    - Output: To List files node.  
    - Failure Modes: None expected; data aggregation.

  - **n8n**  
    - Type: Set  
    - Role: Adds GitHub repo information and marks origin as "n8n" for workflows pulled from n8n.  
    - Configuration:  
      - repo.owner and repo.name from Globals.  
      - origin: "n8n"  
    - Input: From Get many workflows node.  
    - Output: To Loop Over Items node.  
    - Failure Modes: Misconfiguration of repo info.

---

#### 2.3 Fetch GitHub Stored Workflows

- **Overview:** Lists all files in the GitHub backup repository and attempts to retrieve each workflow file's content.
- **Nodes Involved:**  
  - List files  
  - github  
  - Loop Over Items  
  - Get file data  
  - If file too large  
  - Get File  

- **Node Details:**

  - **List files**  
    - Type: GitHub node  
    - Role: Lists all files in the specified GitHub repository.  
    - Configuration: Uses repo.owner and repo.name from Globals node; authentication via OAuth2.  
    - Input: From Workflows node.  
    - Output: To github node.  
    - Failure Modes: GitHub API rate limits, auth errors.

  - **github**  
    - Type: Set  
    - Role: Sets repo info and marks origin as "github". Also attaches the workflows array from Workflows node.  
    - Configuration:  
      - repo.owner and repo.name from Globals.  
      - origin: "github"  
      - workflows: array from Workflows node JSON data.  
    - Input: From List files node.  
    - Output: To Loop Over Items node.  
    - Failure Modes: Data format inconsistencies.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes items in batches to reduce memory usage and manage rate limits.  
    - Configuration: Resets based on context "done" flag.  
    - Input: From n8n and github nodes.  
    - Output: To Execute Workflow and downstream nodes.  
    - Failure Modes: Improper batch sizes may cause memory or timeout issues.

  - **Get file data**  
    - Type: GitHub node  
    - Role: Fetches individual workflow JSON files from GitHub by filename (ID.json).  
    - Configuration: Uses repo info and filePath derived from Switch node JSON ID. OAuth2 authentication.  
    - Input: From Switch node.  
    - Output: To If file too large node.  
    - Failure Modes: File not found (continue on fail enabled), network issues.

  - **If file too large**  
    - Type: If  
    - Role: Checks if the file content is empty or if an error occurred (likely due to file size limits), to decide fetching strategy.  
    - Configuration: Checks if content is empty and error does not exist.  
    - Input: From Get file data node.  
    - Output:  
      - True: To Get File node for alternative fetching.  
      - False: To Merge node.  
    - Failure Modes: Misinterpretation of file content or errors.

  - **Get File**  
    - Type: HTTP Request  
    - Role: Alternative method to get file content via direct download URL if GitHub API returns no content due to file size.  
    - Configuration: URL dynamically set from JSON download_url attribute.  
    - Input: From If file too large node.  
    - Output: To Merge node.  
    - Failure Modes: HTTP errors, download failures.

---

#### 2.4 Compare Workflows

- **Overview:** Compares the current n8n workflow JSON with the corresponding GitHub stored workflow JSON to detect if it is new, different, or unchanged.
- **Nodes Involved:**  
  - Merge  
  - isDiffOrNew  
  - Check Status  
  - File is new  
  - File is different  
  - Same file - Do nothing  

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines the GitHub file JSON and n8n workflow JSON for comparison.  
    - Configuration: Default merge with inputs from Get File and n8n nodes.  
    - Input: From Get File and n8n nodes.  
    - Output: To isDiffOrNew node.  
    - Failure Modes: Data mismatch if inputs are missing.

  - **isDiffOrNew**  
    - Type: Code  
    - Role: Custom JavaScript to compare two JSON objects representing workflows.  
    - Configuration:  
      - Orders JSON keys for consistent comparison.  
      - Decodes base64 content from GitHub file if present.  
      - Compares decoded GitHub workflow to current n8n workflow.  
      - Sets github_status to "same", "different", or "new".  
      - Stores stringified n8n workflow JSON if different or new.  
    - Input: From Merge node.  
    - Output: To Check Status node.  
    - Failure Modes: JSON parse errors, base64 decoding errors.

  - **Check Status**  
    - Type: Switch  
    - Role: Routes the flow based on github_status field set by isDiffOrNew.  
    - Configuration:  
      - Outputs: new, different, same.  
    - Input: From isDiffOrNew node.  
    - Output:  
      - "new" to File is new node.  
      - "different" to File is different node.  
      - "same" to Same file - Do nothing node.  
    - Failure Modes: Misrouted data if github_status missing or malformed.

  - **File is new**  
    - Type: NoOp  
    - Role: Placeholder to trigger creation of a new file in GitHub.  
    - Input: From Check Status node.  
    - Output: To Create new file node.

  - **File is different**  
    - Type: NoOp  
    - Role: Placeholder to trigger editing of an existing file in GitHub.  
    - Input: From Check Status node.  
    - Output: To Edit existing file node.

  - **Same file - Do nothing**  
    - Type: NoOp  
    - Role: Placeholder to end flow when no update needed.  
    - Input: From Check Status node.  
    - Output: To Return node.

---

#### 2.5 Update GitHub Repository

- **Overview:** Performs the actual GitHub repository file management—creating new files or editing existing ones based on previous comparison.
- **Nodes Involved:**  
  - Create new file  
  - Edit existing file  

- **Node Details:**

  - **Create new file**  
    - Type: GitHub node  
    - Role: Creates a new file in the GitHub repo with the workflow JSON content.  
    - Configuration:  
      - filename: `${id}.json` (workflow ID)  
      - repo and owner from Switch node JSON.  
      - content from isDiffOrNew node (`n8n_data_stringy`).  
      - commit message includes workflow name and status.  
      - Authentication: OAuth2.  
    - Input: From File is new node.  
    - Output: To Return node.  
    - Failure Modes: Permission issues, file conflicts.

  - **Edit existing file**  
    - Type: GitHub node  
    - Role: Updates existing workflow file in GitHub with new content.  
    - Configuration: Similar to Create new file, but operation set to "edit".  
    - Input: From File is different node.  
    - Output: To Return node.  
    - Failure Modes: Conflicts, permission errors.

---

#### 2.6 Deleted Workflow Detection and Removal

- **Overview:** Detects workflows that have been deleted from n8n but still exist on GitHub, then deletes the corresponding files from GitHub.
- **Nodes Involved:**  
  - Switch  
  - isDeleted  
  - If  
  - Delete a file  

- **Node Details:**

  - **Switch**  
    - Type: Switch  
    - Role: Routes items based on origin field ("n8n" or "github").  
    - Configuration:  
      - Output "n8n" if origin equals "n8n".  
      - Output "github" if origin equals "github".  
    - Input: From Execute Workflow Trigger node.  
    - Output:  
      - "n8n" to Loop Over Items node (continue processing).  
      - "github" to isDeleted node (check for deletions).  
    - Failure Modes: Misclassification if origin field missing or malformed.

  - **isDeleted**  
    - Type: Code  
    - Role: Checks if the GitHub file corresponds to an existing n8n workflow; sets flag if deleted.  
    - Configuration: Compares workflow ID from GitHub filename to IDs in n8n workflows array.  
    - Input: From Switch node ("github" output).  
    - Output: To If node.  
    - Failure Modes: Incorrect parsing of filename or workflows data.

  - **If**  
    - Type: If  
    - Role: Branches flow based on isDeleted boolean: true leads to deletion, false skips.  
    - Configuration: Checks if isDeleted is true.  
    - Input: From isDeleted node.  
    - Output:  
      - True: To Delete a file node.  
      - False: To Return node.  
    - Failure Modes: Logic failure if isDeleted not properly set.

  - **Delete a file**  
    - Type: GitHub node  
    - Role: Deletes the workflow file from the GitHub repo corresponding to a deleted n8n workflow.  
    - Configuration:  
      - repo and owner from Switch node JSON.  
      - filename from Switch node JSON name.  
      - commit message indicates deletion.  
      - Authentication: OAuth2.  
    - Input: From If node (true output).  
    - Output: To Return node.  
    - Failure Modes: Permission errors, file not found.

---

#### 2.7 Subworkflow Execution and Loop Management

- **Overview:** Manages batch processing of workflows to avoid memory overload by looping through items and invoking self-execution as a subworkflow.
- **Nodes Involved:**  
  - Loop Over Items  
  - Execute Workflow  
  - Execute Workflow Trigger  
  - Return  

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes workflows in batches; resets based on context flag "done".  
    - Configuration: Default batch size and reset management.  
    - Input: From n8n and github nodes.  
    - Output:  
      - First output: For further processing (e.g., to GitHub create/edit).  
      - Second output: To Execute Workflow node for subworkflow invocation.  
    - Failure Modes: Improper batch size may cause performance issues.

  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Invokes this workflow itself as a subworkflow to manage memory and batch processing.  
    - Configuration:  
      - Mode: each item.  
      - Workflow ID: Current workflow's ID for self-invocation.  
      - Input: Passes current item data.  
    - Input: From Loop Over Items node.  
    - Output: To Loop Over Items node (to continue batch processing).  
    - Failure Modes: Infinite loops or recursion if reset not handled correctly.

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Receives input for the subworkflow execution and routes based on origin.  
    - Configuration: Input source passthrough.  
    - Input: From Loop Over Items (via Execute Workflow).  
    - Output: To Switch node.  
    - Failure Modes: Input mismatches.

  - **Return**  
    - Type: Set  
    - Role: Ends the flow with a Done flag.  
    - Configuration: Sets boolean field "Done" to true.  
    - Input: From various terminal nodes (e.g., file creation, deletion, no-op).  
    - Output: Ends workflow execution.  
    - Failure Modes: None.

---

### 3. Summary Table

| Node Name              | Node Type              | Functional Role                              | Input Node(s)                          | Output Node(s)                     | Sticky Note                                                                                                     |
|------------------------|------------------------|----------------------------------------------|--------------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger         | Starts workflow manually                      | None                                 | Globals                          |                                                                                                                 |
| Schedule Trigger        | Schedule Trigger       | Starts workflow automatically daily at 7AM  | None                                 | Globals                          |                                                                                                                 |
| Globals                | Set                    | Sets GitHub repo owner and repo name          | On clicking 'execute', Schedule Trigger | Get many workflows               |                                                                                                                 |
| Get many workflows      | n8n API node           | Fetches all workflows from n8n instance       | Globals                              | Workflows, n8n                   |                                                                                                                 |
| Workflows               | Aggregate              | Aggregates workflows into an array             | Get many workflows                   | List files                      |                                                                                                                 |
| n8n                     | Set                    | Adds repo info and origin "n8n"                 | Get many workflows                   | Loop Over Items                 |                                                                                                                 |
| List files              | GitHub                 | Lists files in GitHub repo                       | Workflows                           | github                         |                                                                                                                 |
| github                  | Set                    | Adds repo info and origin "github", workflows array | List files                        | Loop Over Items                 |                                                                                                                 |
| Loop Over Items         | Split In Batches       | Processes workflows in batches                   | n8n, github                        | Execute Workflow, downstream nodes |                                                                                                                 |
| Execute Workflow        | Execute Workflow       | Invokes workflow as subworkflow                   | Loop Over Items                    | Loop Over Items                 | The workflow calls itself using a subworkflow, to help reduce memory usage.                                     |
| Execute Workflow Trigger| Execute Workflow Trigger| Receives input for subworkflow execution        | Execute Workflow                   | Switch                         |                                                                                                                 |
| Switch                  | Switch                 | Routes items based on origin                      | Execute Workflow Trigger            | Get file data, isDeleted        |                                                                                                                 |
| Get file data           | GitHub                 | Retrieves individual workflow file from GitHub  | Switch                             | If file too large               |                                                                                                                 |
| If file too large       | If                     | Checks if file content is empty or error exists | Get file data                      | Get File, Merge                 |                                                                                                                 |
| Get File                | HTTP Request           | Downloads file content from GitHub URL          | If file too large                  | Merge                          |                                                                                                                 |
| Merge                   | Merge                  | Combines GitHub file data and n8n workflow data | Get File, n8n                     | isDiffOrNew                    |                                                                                                                 |
| isDiffOrNew             | Code                   | Compares workflows, determines status            | Merge                             | Check Status                   |                                                                                                                 |
| Check Status            | Switch                 | Routes based on comparison status                 | isDiffOrNew                      | File is new/different/same      |                                                                                                                 |
| File is new             | NoOp                   | Triggers creation of new file                      | Check Status                     | Create new file                |                                                                                                                 |
| File is different       | NoOp                   | Triggers editing of existing file                  | Check Status                     | Edit existing file             |                                                                                                                 |
| Same file - Do nothing  | NoOp                   | Ends flow when no update is needed                 | Check Status                     | Return                        |                                                                                                                 |
| Create new file         | GitHub                 | Creates new workflow file in GitHub repo           | File is new                     | Return                        |                                                                                                                 |
| Edit existing file      | GitHub                 | Edits existing workflow file in GitHub repo        | File is different               | Return                        |                                                                                                                 |
| isDeleted               | Code                   | Checks if GitHub file corresponds to deleted workflow | Switch                          | If                            |                                                                                                                 |
| If                      | If                     | Branches flow to delete or skip file                | isDeleted                      | Delete a file, Return          |                                                                                                                 |
| Delete a file           | GitHub                 | Deletes workflow file from GitHub repo              | If                            | Return                        |                                                                                                                 |
| Return                  | Set                    | Sets "Done" flag to true, ends flow                  | Multiple terminal nodes         | None                          |                                                                                                                 |
| Sticky Note1            | Sticky Note            | Instruction to edit repo.owner and repo.name        | None                           | None                          | ## Edit this node 👇                                                                                              |
| Sticky Note2            | Sticky Note            | Indicates subworkflow section                        | None                           | None                          | ## Subworkflow                                                                                                   |
| Sticky Note3            | Sticky Note            | Instruction to edit Globals node                      | None                           | None                          | ## Backup to GitHub \nThis workflow will backup all instance workflows to GitHub and also deleted it if was deleted in n8n.\n\nThe files are saved `ID.json` for the filename.\n\n### Setup\nOpen `Globals` node and update the values below 👇\n\n- **repo.owner:** your Github username\n- **repo.name:** the name of your repository\n\n\nIf your username was `john-doe` and your repository was called `n8n-backups`:\n\n- repo.owner - john-doe\n- repo.name - n8n-backups\n\n\nThe workflow calls itself using a subworkflow, to help reduce memory usage. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Create **Manual Trigger** node named `On clicking 'execute'`.
   - Create **Schedule Trigger** node named `Schedule Trigger`, configure to run daily at 7:00 AM.
   - Connect both triggers to the next node.

2. **Set Global Variables:**

   - Create a **Set** node named `Globals`.
   - Add fields:  
     - `repo.owner` (string): Enter your GitHub username (e.g., "john-doe").  
     - `repo.name` (string): Enter your GitHub repository name (e.g., "n8n-backup").
   - Connect both triggers to `Globals`.

3. **Fetch Workflows from n8n:**

   - Create **n8n** API node named `Get many workflows`.
   - Configure with credentials to access your n8n instance.
   - Leave filters empty to fetch all workflows.
   - Connect `Globals` node output to this node.

4. **Aggregate Workflow Data:**

   - Create an **Aggregate** node named `Workflows`.
   - Set to aggregate all item data (`aggregateAllItemData`).
   - Connect `Get many workflows` output to this node.

5. **Set Origin and Repo Info for n8n Workflows:**

   - Create **Set** node named `n8n`.
   - Set fields:  
     - `repo.owner`: `={{ $('Globals').item.json.repo.owner }}`  
     - `repo.name`: `={{ $('Globals').item.json.repo.name }}`  
     - `origin`: `"n8n"`  
   - Enable "Include Other Fields" to pass through workflow data.
   - Connect `Get many workflows` output to this node.

6. **List Files in GitHub Repo:**

   - Create **GitHub** node named `List files`.
   - Configure:  
     - Owner: `={{ $('Globals').item.json.repo.owner }}`  
     - Repository: `={{ $('Globals').item.json.repo.name }}`  
     - Resource: File  
     - Operation: List  
     - Authentication: OAuth2 with GitHub credentials.  
   - Connect `Workflows` node to `List files`.

7. **Set Origin and Repo Info for GitHub Files:**

   - Create **Set** node named `github`.
   - Set fields:  
     - `repo.owner`: `={{ $('Globals').item.json.repo.owner }}`  
     - `repo.name`: `={{ $('Globals').item.json.repo.name }}`  
     - `origin`: `"github"`  
     - `workflows`: `={{ $('Workflows').item.json.data }}` (pass workflows array)  
   - Enable "Include Other Fields."  
   - Connect `List files` node to this node.

8. **Batch Processing Setup:**

   - Create **Split In Batches** node named `Loop Over Items`.
   - Configure:  
     - Reset: `={{ $node["Loop Over Items"].context["done"] }}` (to allow looping)  
   - Connect both `n8n` and `github` nodes outputs to `Loop Over Items`.

9. **Subworkflow Invocation:**

   - Create **Execute Workflow** node named `Execute Workflow`.
   - Set mode to `each`.
   - Set workflow ID to this current workflow's ID (self-invocation).
   - Input mode: pass current item.
   - Connect second output of `Loop Over Items` to this node.
   - Connect `Execute Workflow` output back to first input of `Loop Over Items` (loop continuation).

10. **Subworkflow Entry Point:**

    - Create **Execute Workflow Trigger** node named `Execute Workflow Trigger`.
    - Set input source to passthrough.
    - Connect output of `Execute Workflow` to this node.

11. **Route by Origin:**

    - Create **Switch** node named `Switch`.
    - Add two rules:  
      - Output "n8n" if `{{$json.origin}}` equals "n8n"  
      - Output "github" if `{{$json.origin}}` equals "github"  
    - Connect `Execute Workflow Trigger` output to `Switch`.

12. **Fetch GitHub File Data:**

    - Create **GitHub** node named `Get file data`.
    - Configure:  
      - Owner: `={{ $json.repo.owner }}`  
      - Repository: `={{ $json.repo.name }}`  
      - File Path: `={{ $('Switch').item.json.id }}.json` (workflow ID + .json)  
      - Operation: Get  
      - Authentication: OAuth2  
      - Continue On Fail: Enabled  
      - Always Output Data: Enabled  
    - Connect "n8n" output of `Switch` to this node.

13. **Check for Large File Handling:**

    - Create **If** node named `If file too large`.
    - Condition:  
      - Check if content is empty string AND  
      - Error property does not exist  
    - True Output: Connect to `Get File` node.  
    - False Output: Connect to `Merge` node.

14. **Alternative File Download:**

    - Create **HTTP Request** node named `Get File`.
    - Set method GET.  
    - URL: `={{ $json.download_url }}` (from GitHub file metadata).  
    - Connect true output of `If file too large` to this node.

15. **Merge GitHub File and n8n Workflow:**

    - Create **Merge** node named `Merge`.
    - Configure to merge inputs by index.  
    - Connect outputs of `Get File` and `n8n` nodes to this node.

16. **Compare Workflows:**

    - Create **Code** node named `isDiffOrNew`.
    - Paste JavaScript code that:  
      - Orders JSON keys for consistent comparison  
      - Decodes base64 content if present  
      - Compares GitHub and n8n workflows  
      - Sets `github_status` to "same", "different", or "new"  
      - Stores stringified workflow JSON if needed  
    - Connect output of `Merge` to this node.

17. **Switch on Comparison Result:**

    - Create **Switch** node named `Check Status`.
    - Rules based on `{{$json.github_status}}`:  
      - "new" → to `File is new`  
      - "different" → to `File is different`  
      - "same" → to `Same file - Do nothing`  
    - Connect `isDiffOrNew` output to this node.

18. **Create or Edit GitHub Files:**

    - Create **NoOp** nodes `File is new`, `File is different`, `Same file - Do nothing` as placeholders.
    - Connect outputs of `Check Status` accordingly:  
      - `File is new` → `Create new file`  
      - `File is different` → `Edit existing file`  
      - `Same file - Do nothing` → `Return`  

19. **Create New File in GitHub:**

    - Create **GitHub** node named `Create new file`.
    - Configure:  
      - Owner: `={{ $('Switch').item.json.repo.owner }}`  
      - Repository: `={{ $('Switch').item.json.repo.name }}`  
      - File Path: `={{ $('Switch').first().json.id }}.json`  
      - Content: `={{ $('isDiffOrNew').item.json["n8n_data_stringy"] }}`  
      - Commit Message: `={{ $('Switch').first().json.name }} ({{$json.github_status}})`  
      - Operation: Create  
      - Authentication: OAuth2  
    - Connect `File is new` node to this node.  
    - Connect output to `Return`.

20. **Edit Existing File in GitHub:**

    - Create **GitHub** node named `Edit existing file`.
    - Configuration similar to `Create new file` but operation set to "edit".
    - Connect `File is different` node to this node.
    - Connect output to `Return`.

21. **Deleted Workflow Detection:**

    - Connect "github" output of `Switch` node to **Code** node named `isDeleted`.
    - Code compares GitHub file name (workflow ID) against current n8n workflows array.
    - Output boolean `isDeleted` true if file corresponds to deleted workflow.

22. **Conditional Deletion Check:**

    - Create **If** node named `If`.
    - Condition: `{{$json.isDeleted}}` is true.
    - True output connects to `Delete a file`.
    - False output connects to `Return`.

23. **Delete File from GitHub:**

    - Create **GitHub** node named `Delete a file`.
    - Configure:  
      - Owner: `={{ $('Switch').item.json.repo.owner }}`  
      - Repository: `={{ $('Switch').item.json.repo.name }}`  
      - File Path: `={{ $('Switch').first().json.name }}`  
      - Commit Message: `={{ $('Switch').first().json.name }} (deleted)`  
      - Operation: Delete  
      - Authentication: OAuth2  
    - Connect `If` node (true) output to this node.  
    - Connect output to `Return`.

24. **Return Node to End Flow:**

    - Create **Set** node named `Return`.
    - Add boolean field "Done" set to true.
    - Connect all terminal nodes (file creation, editing, deletion, no-op) to this node.

25. **Add Sticky Notes as Comments:**

    - Add three sticky notes per original workflow for guidance:  
      - Backup to GitHub instructions and setup.  
      - Main workflow loop explanation.  
      - Subworkflow explanation and memory management.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow backs up all n8n workflows to GitHub and deletes GitHub files if the workflow was deleted in n8n. Files are saved as `ID.json` where ID is the workflow ID. Setup requires updating the Globals node with your GitHub username and repo name. The workflow uses a self-calling subworkflow to reduce memory usage. | Sticky Note content in workflow.             |
| GitHub OAuth2 credentials are required and should be created with permission to read/write repository files.                                                                                                                                 | GitHub OAuth2 API credential setup.          |
| The workflow handles large files by detecting empty content from GitHub API and fetching raw content via the download URL.                                                                                                                   | Large file handling block.                     |
| Batch processing and subworkflow execution help prevent memory overload and manage processing of multiple workflows efficiently.                                                                                                            | Loop Over Items and Execute Workflow nodes.  |
| This workflow is designed for n8n version 0.200+ due to usage of certain node types and features (e.g., Execute Workflow Trigger, advanced Switch conditions).                                                                               | Version requirement note.                      |

---

This document provides a comprehensive and structured description of the Automated n8n Workflow Backup to GitHub with Deletion Tracking workflow, enabling users and AI agents to understand, reproduce, and maintain it effectively.