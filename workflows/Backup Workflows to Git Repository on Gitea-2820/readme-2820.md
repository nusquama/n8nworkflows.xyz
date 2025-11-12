Backup Workflows to Git Repository on Gitea

https://n8nworkflows.xyz/workflows/backup-workflows-to-git-repository-on-gitea-2820


# Backup Workflows to Git Repository on Gitea

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows to a Git repository hosted on a Gitea server. It is designed to run on a scheduled interval, retrieve all workflows via the n8n API, encode them into a Git-compatible format, and commit them to a specified Gitea repository. The workflow ensures that only changed workflows are updated in the repository, preventing unnecessary commits.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Initialization:** Starts the workflow on a schedule and sets global configuration variables.
- **1.2 Fetch Workflows:** Retrieves all workflows from the n8n instance via API.
- **1.3 Iterate Over Workflows:** Processes each workflow individually using a batch/split node.
- **1.4 Check Repository for Existing Files:** For each workflow, checks if a corresponding file exists in the Gitea repository.
- **1.5 Prepare Workflow Data:** Encodes workflow JSON data to base64 for Git storage.
- **1.6 Compare & Commit Changes:** Determines if the workflow has changed compared to the repository version, then either updates or creates the file in Gitea.
- **1.7 Loop Control & Completion:** Manages iteration and finalizes the process.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Initialization

- **Overview:** This block triggers the workflow on a recurring schedule and sets essential global variables that define the Gitea repository details.
- **Nodes Involved:** `Schedule Trigger`, `Globals`, `n8n`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Initiates workflow every 45 minutes.
    - Configuration: Interval set to 45 minutes.
    - Inputs: None (trigger node)
    - Outputs: Connects to `Globals`
    - Edge cases: Misconfiguration of interval could cause unexpected frequency.

  - **Globals**
    - Type: Set Node
    - Role: Defines global variables for repository URL, name, and owner.
    - Configuration: Sets three string variables:
      - `repo.url` = `https://git.vdm.dev` (default, user should update)
      - `repo.name` = `workflows`
      - `repo.owner` = `n8n`
    - Inputs: From `Schedule Trigger`
    - Outputs: Connects to `n8n`
    - Edge cases: Incorrect URLs or repository names will cause API failures.

  - **n8n**
    - Type: n8n API Node
    - Role: Fetches all workflows from the n8n instance.
    - Configuration: Uses stored credentials for API access.
    - Inputs: From `Globals`
    - Outputs: Connects to `ForEach`
    - Edge cases: API auth errors, network timeouts.

---

#### 2.2 Fetch Workflows & Iterate

- **Overview:** Retrieves all workflows and splits them into individual items for processing.
- **Nodes Involved:** `n8n`, `ForEach`
- **Node Details:**

  - **n8n**
    - (Described above)

  - **ForEach**
    - Type: SplitInBatches
    - Role: Processes workflows one by one.
    - Configuration: Default batch size (1), no special options.
    - Inputs: From `n8n`
    - Outputs: Two outputs:
      - Main output (empty, unused)
      - Secondary output to `GetGitea` node for further processing.
    - Edge cases: Large number of workflows could slow processing.

---

#### 2.3 Check Repository for Existing Files

- **Overview:** For each workflow, checks if a corresponding JSON file exists in the Gitea repository.
- **Nodes Involved:** `GetGitea`, `Exist`, `SetDataUpdateNode`, `SetDataCreateNode`
- **Node Details:**

  - **GetGitea**
    - Type: HTTP Request
    - Role: Queries Gitea API for the workflow file content.
    - Configuration:
      - URL constructed dynamically from globals and current workflow name.
      - Authentication: Gitea Token via HTTP Header.
      - On error: Continue regular output (handles 404 if file not found).
    - Inputs: From `ForEach`
    - Outputs: Connects to `Exist`
    - Edge cases: 404 if file does not exist; network or auth errors.

  - **Exist**
    - Type: If Node
    - Role: Determines if the file exists based on API response.
    - Configuration:
      - Checks if `error` property exists and is not 404.
      - If file exists → true branch to `SetDataUpdateNode`
      - If file does not exist → false branch to `SetDataCreateNode`
    - Inputs: From `GetGitea`
    - Outputs: Two outputs to `SetDataUpdateNode` and `SetDataCreateNode`
    - Edge cases: Unexpected API errors.

  - **SetDataUpdateNode**
    - Type: Set Node
    - Role: Prepares data for updating existing workflow file.
    - Configuration:
      - Assigns current workflow JSON item to a new field `item`.
    - Inputs: From `Exist` (true branch)
    - Outputs: Connects to `Base64EncodeUpdate`
    - Edge cases: Data format issues.

  - **SetDataCreateNode**
    - Type: Set Node
    - Role: Prepares data for creating a new workflow file.
    - Configuration:
      - Assigns current workflow JSON item to a new field `item`.
    - Inputs: From `Exist` (false branch)
    - Outputs: Connects to `Base64EncodeCreate`
    - Edge cases: Data format issues.

---

#### 2.4 Prepare Workflow Data (Base64 Encoding)

- **Overview:** Converts workflow JSON data into base64-encoded strings suitable for Git storage.
- **Nodes Involved:** `Base64EncodeUpdate`, `Base64EncodeCreate`
- **Node Details:**

  - **Base64EncodeUpdate**
    - Type: Code (Python)
    - Role: Encodes workflow JSON to base64 for update operation.
    - Configuration:
      - Converts input JS object to Python dict.
      - Extracts workflow JSON.
      - Serializes to pretty-printed JSON string.
      - Encodes to base64 string.
      - Returns object with base64 string in `item`.
    - Inputs: From `SetDataUpdateNode`
    - Outputs: Connects to `Changed`
    - Edge cases: Encoding errors, malformed JSON.

  - **Base64EncodeCreate**
    - Type: Code (Python)
    - Role: Encodes workflow JSON to base64 for create operation.
    - Configuration: Same as `Base64EncodeUpdate`.
    - Inputs: From `SetDataCreateNode`
    - Outputs: Connects to `PostGitea`
    - Edge cases: Encoding errors, malformed JSON.

---

#### 2.5 Compare & Commit Changes

- **Overview:** Compares the base64-encoded content with the repository version and commits changes if needed.
- **Nodes Involved:** `Changed`, `PutGitea`, `PostGitea`
- **Node Details:**

  - **Changed**
    - Type: If Node
    - Role: Checks if the base64 content differs from the repository content.
    - Configuration:
      - Compares `item` from encoded data with `content` from `GetGitea`.
      - If different → true branch to `PutGitea`
      - If same → false branch loops back to `ForEach` for next item.
    - Inputs: From `Base64EncodeUpdate`
    - Outputs: Two outputs to `PutGitea` and `ForEach`
    - Edge cases: Comparison errors, encoding mismatches.

  - **PutGitea**
    - Type: HTTP Request
    - Role: Updates existing workflow file in Gitea repository.
    - Configuration:
      - Method: PUT
      - URL: Constructed dynamically using globals and file name.
      - Body parameters:
        - `content`: base64-encoded workflow JSON
        - `sha`: current file SHA from `GetGitea`
      - Authentication: Gitea Token via HTTP Header.
    - Inputs: From `Changed`
    - Outputs: Connects back to `ForEach`
    - Edge cases: API errors, auth failures, SHA conflicts.

  - **PostGitea**
    - Type: HTTP Request
    - Role: Creates new workflow file in Gitea repository.
    - Configuration:
      - Method: POST
      - URL: Constructed dynamically using globals and file name.
      - Body parameters:
        - `content`: base64-encoded workflow JSON
      - Authentication: Gitea Token via HTTP Header.
    - Inputs: From `Base64EncodeCreate`
    - Outputs: Connects back to `ForEach`
    - Edge cases: API errors, auth failures.

---

#### 2.6 Sticky Notes

Sticky notes provide user guidance and explanations at various points:

- Setup instructions for global variables and credentials.
- Notes on checking file existence.
- Confirmation that workflow changes are committed.
- Reminders about the scheduled trigger and API authentication.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                   |
|--------------------|---------------------|----------------------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger    | Triggers workflow every 45 minutes     | None                  | Globals                |                                                                                               |
| Globals            | Set                 | Sets global variables for repo details | Schedule Trigger       | n8n                    | Set variables                                                                                |
| n8n                | n8n API             | Fetches all workflows from n8n          | Globals                | ForEach                | Get all workflows                                                                            |
| ForEach            | SplitInBatches      | Processes workflows one by one          | n8n                    | GetGitea (secondary)   |                                                                                               |
| GetGitea           | HTTP Request        | Checks if workflow file exists in repo  | ForEach                | Exist                  | Check if file exists in the repository                                                      |
| Exist              | If                  | Determines if file exists                | GetGitea                | SetDataUpdateNode, SetDataCreateNode | Check if there are any changes in the workflow                                               |
| SetDataUpdateNode  | Set                 | Prepares data for updating file         | Exist (true branch)     | Base64EncodeUpdate     |                                                                                               |
| SetDataCreateNode  | Set                 | Prepares data for creating file         | Exist (false branch)    | Base64EncodeCreate     | Create a new file for the workflow                                                          |
| Base64EncodeUpdate | Code (Python)       | Encodes workflow JSON to base64 (update)| SetDataUpdateNode       | Changed                |                                                                                               |
| Base64EncodeCreate | Code (Python)       | Encodes workflow JSON to base64 (create)| SetDataCreateNode       | PostGitea              |                                                                                               |
| Changed            | If                  | Checks if workflow content changed      | Base64EncodeUpdate      | PutGitea, ForEach      | Check if there are any changes in the workflow                                               |
| PutGitea           | HTTP Request        | Updates existing workflow file in repo  | Changed (true branch)   | ForEach                | Workflow changes committed to the repository                                                |
| PostGitea          | HTTP Request        | Creates new workflow file in repo       | Base64EncodeCreate      | ForEach                | Workflow changes committed to the repository                                                |
| Sticky Note        | Sticky Note         | Notes on committing workflow changes    | None                   | None                   | Workflow changes committed to the repository                                                |
| Sticky Note1       | Sticky Note         | Notes on checking for workflow changes  | None                   | None                   | Check if there are any changes in the workflow                                              |
| Sticky Note2       | Sticky Note         | Notes on creating new workflow file     | None                   | None                   | Create a new file for the workflow                                                          |
| Sticky Note3       | Sticky Note         | Notes on checking file existence         | None                   | None                   | Check if file exists in the repository                                                      |
| Sticky Note4       | Sticky Note         | Setup guide for the entire workflow     | None                   | None                   | Detailed setup guide with credentials and variables                                         |
| Sticky Note5       | Sticky Note         | Notes on getting all workflows           | None                   | None                   | Get all workflows                                                                           |
| Sticky Note6       | Sticky Note         | Notes on setting variables                | None                   | None                   | Set variables                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Set it to run every 45 minutes.
   - No credentials needed.

2. **Create a Set node named "Globals":**
   - Add three string variables:
     - `repo.url` → your Gitea instance URL (e.g., `https://git.vdm.dev`)
     - `repo.name` → repository name (e.g., `workflows`)
     - `repo.owner` → repository owner (e.g., `n8n`)
   - Connect Schedule Trigger output to this node.

3. **Create an n8n API node named "n8n":**
   - Configure to fetch all workflows (`GET /workflows`).
   - Use your n8n API credentials.
   - Connect Globals output to this node.

4. **Create a SplitInBatches node named "ForEach":**
   - Default batch size (1).
   - Connect n8n output to this node.

5. **Create an HTTP Request node named "GetGitea":**
   - Method: GET
   - URL: `={{ $('Globals').item.json.repo.url }}/api/v1/repos/{{ encodeURIComponent($('Globals').item.json.repo.owner) }}/{{ encodeURIComponent($('Globals').item.json.repo.name) }}/contents/{{ encodeURIComponent($json.name) }}.json`
   - Authentication: HTTP Header Auth with Gitea Token credential.
   - On error: Continue regular output.
   - Connect ForEach secondary output to this node.

6. **Create an If node named "Exist":**
   - Condition: Check if `$json.error` does not exist or is not 404.
   - True output: file exists.
   - False output: file does not exist.
   - Connect GetGitea output to this node.

7. **Create two Set nodes:**
   - "SetDataUpdateNode" (for true branch):
     - Assign `item` = current workflow JSON (`={{ $node["ForEach"].json }}`).
     - Connect Exist true output to this node.
   - "SetDataCreateNode" (for false branch):
     - Assign `item` = current workflow JSON (`={{ $node["ForEach"].json }}`).
     - Connect Exist false output to this node.

8. **Create two Code nodes (Python):**
   - "Base64EncodeUpdate" connected from SetDataUpdateNode.
   - "Base64EncodeCreate" connected from SetDataCreateNode.
   - Both nodes run Python code to:
     - Convert input JS object to Python dict.
     - Extract workflow JSON.
     - Serialize to pretty JSON string.
     - Encode to base64 string.
     - Return `{ "item": base64_string }`.

9. **Create an If node named "Changed":**
   - Condition: Check if base64-encoded content (`$json.item`) is not equal to the content from GetGitea (`$('GetGitea').item.json.content`).
   - True output: content changed.
   - False output: content unchanged.
   - Connect Base64EncodeUpdate output to this node.

10. **Create HTTP Request nodes:**
    - "PutGitea" (update existing file):
      - Method: PUT
      - URL: `={{ $('Globals').item.json.repo.url }}/api/v1/repos/{{ $('Globals').item.json.repo.owner }}/{{ $('Globals').item.json.repo.name }}/contents/{{ encodeURIComponent($('GetGitea').item.json.name) }}`
      - Body parameters:
        - `content`: base64 string from Base64EncodeUpdate
        - `sha`: SHA from GetGitea
      - Authentication: HTTP Header Auth with Gitea Token.
      - Connect Changed true output to this node.
    - "PostGitea" (create new file):
      - Method: POST
      - URL: `={{ $('Globals').item.json.repo.url }}/api/v1/repos/{{ $('Globals').item.json.repo.owner }}/{{ $('Globals').item.json.repo.name }}/contents/{{ encodeURIComponent($('ForEach').item.json.name) }}.json`
      - Body parameters:
        - `content`: base64 string from Base64EncodeCreate
      - Authentication: HTTP Header Auth with Gitea Token.
      - Connect Base64EncodeCreate output to this node.

11. **Connect PutGitea and PostGitea outputs back to ForEach node:**
    - This loops the process for the next workflow.

12. **Configure Credentials:**
    - Create Gitea Token credential in n8n Credentials Manager:
      - Name: `Authorization`
      - Value: `Bearer YOUR_PERSONAL_ACCESS_TOKEN` (note the space after "Bearer")
    - Assign this credential to `GetGitea`, `PutGitea`, and `PostGitea` nodes.
    - Assign appropriate API credentials to the `n8n` node for fetching workflows.

13. **Test the workflow manually:**
    - Run once to verify workflows are backed up correctly.
    - Check the Gitea repository for created/updated files.

14. **Activate the Schedule Trigger:**
    - Enable the trigger for automated periodic backups.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Ensure your Gitea personal access token has repo read/write permissions.                                                          | Gitea Settings → Applications → Generate Token                                                  |
| The workflow automatically checks for changes before committing updates to avoid unnecessary commits.                            | Workflow logic in `Changed` node                                                                 |
| For detailed setup instructions, refer to the sticky note attached near the Schedule Trigger node in the workflow editor.        | Sticky Note4 content in the workflow                                                            |
| Reach out on the n8n community forum for support or questions related to this workflow.                                            | https://community.n8n.io                                                                         |
| The base64 encoding Python code handles JS Proxy objects conversion to ensure compatibility between JavaScript and Python objects.| Nodes: Base64EncodeCreate and Base64EncodeUpdate                                                |

---

This documentation provides a full understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.