Automated Workflow Backup to GitHub on Schedule

https://n8nworkflows.xyz/workflows/automated-workflow-backup-to-github-on-schedule-6392


# Automated Workflow Backup to GitHub on Schedule

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows to a GitHub repository on a scheduled interval of every 6 hours (configurable). It is designed for users who want to ensure their workflow definitions are regularly backed up for version control, disaster recovery, or team collaboration.

The workflow’s logic is grouped into these functional blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow every 6 hours automatically.
- **1.2 Export Workflows**: Fetches all existing workflows from the n8n instance via API.
- **1.3 Data Preparation**: Converts the fetched JSON workflows data into binary format suitable for file upload.
- **1.4 GitHub File Management**: Attempts to update the backup file in the GitHub repository; if the file does not exist, creates a new one.
- **1.5 Conditional Logic**: Checks GitHub API response for file existence and routes accordingly.
- **1.6 Final Merge and Create**: Merges data to preserve previous file info if needed and creates the backup file in GitHub.


### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Trigger

- **Overview:**  
Launches the workflow on a fixed schedule, triggering every 6 hours without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Configuration: Set to trigger every 6 hours using the interval schedule (hoursInterval=6).  
    - Input: None (trigger node)  
    - Output: Initiates the workflow chain by triggering the next node "Get All Workflows".  
    - Version Requirement: 1.1 or higher for interval scheduling.  
    - Potential Failures: None typical; depends on n8n server uptime and scheduling engine.  
    - Notes: The backup frequency can be changed by adjusting the hoursInterval parameter.

---

#### 1.2 Export Workflows

- **Overview:**  
Retrieves all current workflows from the n8n instance REST API as a JSON array for backup.

- **Nodes Involved:**  
  - Get All Workflows

- **Node Details:**  
  - **Get All Workflows**  
    - Type: HTTP Request node  
    - Configuration:  
      - Method: GET  
      - URL: `https://your-n8n-instance.com/rest/workflows?limit=9999` (replace with actual endpoint)  
      - Headers:  
        - Accept: application/json  
        - X-N8N-API-KEY: `<n8n-api-key>` (must be replaced with actual API key)  
    - Input: Triggered by Schedule Trigger  
    - Output: JSON payload containing all workflows  
    - Version Requirement: 4.2 or higher for latest HTTP node features  
    - Potential Failures:  
      - Authentication error if API key is invalid or missing  
      - Network errors if n8n instance is inaccessible  
      - API limit or timeout if workflows exceed limit or slow response  
    - Notes: Ensure API key is kept secure; adjust URL to your deployment.

---

#### 1.3 Data Preparation

- **Overview:**  
Transforms the JSON workflows data into binary format with a filename for GitHub upload compatibility.

- **Nodes Involved:**  
  - Move Binary Data

- **Node Details:**  
  - **Move Binary Data**  
    - Type: Move Binary Data node  
    - Configuration:  
      - Mode: jsonToBinary (converts JSON data to binary)  
      - Options:  
        - fileName: `"backup"` (sets filename in binary data)  
        - useRawData: true (uses raw JSON data as input)  
    - Input: JSON data from "Get All Workflows"  
    - Output: Binary data containing the backup file content  
    - Version Requirement: 1 or higher  
    - Potential Failures: Expression evaluation errors if input JSON is malformed  
    - Notes: Prepares data for GitHub file operations.

---

#### 1.4 GitHub File Management

- **Overview:**  
Attempts to edit (update) the backup file `backup.json` in the specified GitHub repository. If the file does not exist, the workflow branches to create it anew.

- **Nodes Involved:**  
  - Edit a file  
  - Create a file  
  - If file not exits? (conditional)  
  - Get Previous FIle back (merge)  

- **Node Details:**  
  - **Edit a file**  
    - Type: GitHub node  
    - Operation: Edit a file in GitHub repository  
    - Configuration:  
      - Owner: GitHub username (credential-linked, dynamic via expression)  
      - Repository: `n8n_backup` (select from list)  
      - File Path: `backup.json`  
      - Commit Message: Dynamic, formatted as `n8n_backup_YYYY-MM-DD hh` using current timestamp  
      - Binary Data: true (uploads binary file content)  
      - Credentials: GitHub OAuth2 credentials configured separately  
    - Input: Binary data from "Move Binary Data"  
    - Output: GitHub API response  
    - OnError: Continue (allows workflow to check error and branch)  
    - Potential Failures:  
      - Authentication errors if token invalid or expired  
      - File not found error triggers fallback logic  
      - API rate limits from GitHub if called too frequently  
    - Notes: The error continuation setting is critical for fallback logic.

  - **If file not exits?**  
    - Type: If node  
    - Configuration: Checks if the error message from "Edit a file" contains `"The resource you are requesting could not be found"` indicating file missing  
    - Input: Output from "Edit a file"  
    - Output:  
      - True branch if file missing (routes to creation)  
      - False branch (implicit) if file exists or other error  
    - Potential Failures: Expression mismatch if error format changes  

  - **Get Previous FIle back**  
    - Type: Merge node  
    - Configuration:  
      - Mode: Combine in enrichInput1 join mode  
      - Fields to match: `data` (to merge file data for creation step)  
    - Input:  
      - Primary: Binary data from "Move Binary Data"  
      - Secondary: Output from "If file not exits?" true branch  
    - Output: Merged data combining new backup data with previous file info if available  
    - Potential Failures: Merge conflicts if data structure varies  

  - **Create a file**  
    - Type: GitHub node  
    - Operation: Create a new file in GitHub repository  
    - Configuration:  
      - Owner: same as "Edit a file"  
      - Repository: `n8n_backup`  
      - File Path: `backup.json`  
      - Commit Message: Same timestamp format as edit  
      - Binary Data: true  
      - Credentials: GitHub OAuth2  
    - Input: Output from "Get Previous FIle back"  
    - Output: GitHub API response confirming file creation  
    - Potential Failures: Authentication errors, API rate limits  
    - Notes: Executes only if file did not exist previously.

---

#### 1.5 Conditional Logic

- **Overview:**  
Handles the decision-making process based on GitHub response to ensure the backup file is either updated or created without failure.

- **Nodes Involved:**  
  - If file not exits?  
  - Get Previous FIle back  

- **Node Details:** Covered above under GitHub File Management.

---

#### 1.6 Notes and Documentation

- **Overview:**  
Contains sticky notes that explain the workflow purpose, setup instructions, and usage recommendations.

- **Nodes Involved:**  
  - Sticky Note7  
  - Sticky Note2  
  - Sticky Note

- **Node Details:**  
  - Sticky Note7: Highlights backup interval is every 6 hours (modifiable).  
  - Sticky Note2: Detailed project description, benefits, setup steps, use cases, and pro tips (see section 5 for full content).  
  - Sticky Note: Explains logic checking if file exists before creating a new one.  
  - All are for user guidance and do not affect workflow logic.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                    | Input Node(s)        | Output Node(s)             | Sticky Note                                                                                                            |
|---------------------|-----------------------|----------------------------------|----------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger       | Trigger workflow every 6 hours   | None                 | Get All Workflows           | ## Backup every 6 Hour (can be changed as per requirement)                                                             |
| Get All Workflows   | HTTP Request          | Fetch all n8n workflows JSON     | Schedule Trigger     | Move Binary Data            | See detailed project description and usage instructions in Sticky Note2                                                |
| Move Binary Data    | Move Binary Data       | Convert JSON to binary data      | Get All Workflows    | Edit a file, Get Previous FIle back | See detailed project description and usage instructions in Sticky Note2                                                |
| Edit a file        | GitHub                 | Update backup file in GitHub     | Move Binary Data     | If file not exits?          | ## Check if File is present or not else it will create file a new file based on error                                   |
| If file not exits?  | If                     | Check if backup file exists      | Edit a file          | Get Previous FIle back      | ## Check if File is present or not else it will create file a new file based on error                                   |
| Get Previous FIle back | Merge                | Merge previous file data for create | Move Binary Data, If file not exits? | Create a file              | ## Check if File is present or not else it will create file a new file based on error                                   |
| Create a file       | GitHub                 | Create backup file if missing    | Get Previous FIle back | None                      | ## Check if File is present or not else it will create file a new file based on error                                   |
| Sticky Note7        | Sticky Note            | Notes on backup interval         | None                 | None                       | ## Backup every 6 Hour (can be changed as per requirement)                                                             |
| Sticky Note2        | Sticky Note            | Detailed workflow overview       | None                 | None                       | # 🚀 Automated n8n Workflow Backup to GitHub for schedule Interval ... (full content in section 5)                      |
| Sticky Note         | Sticky Note            | Explains conditional file logic  | None                 | None                       | ## Check if File is present or not else it will create file a new file based on error                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure: Set interval to trigger every 6 hours (hoursInterval=6).

2. **Create HTTP Request Node "Get All Workflows"**  
   - Connect Schedule Trigger → Get All Workflows  
   - Configure:  
     - Method: GET  
     - URL: `https://your-n8n-instance.com/rest/workflows?limit=9999` (replace with your n8n API endpoint)  
     - Headers:  
       - Accept: application/json  
       - X-N8N-API-KEY: `<your-n8n-api-key>` (replace with your actual API key)  

3. **Create Move Binary Data Node**  
   - Connect Get All Workflows → Move Binary Data  
   - Configure:  
     - Mode: jsonToBinary  
     - Options:  
       - fileName: `"backup"`  
       - useRawData: true  

4. **Create GitHub Node "Edit a file"**  
   - Connect Move Binary Data → Edit a file  
   - Configure:  
     - Resource: File  
     - Operation: Edit  
     - Owner: Enter your GitHub username (or set to use credential reference)  
     - Repository: Select or enter your backup repository (e.g., `n8n_backup`)  
     - File Path: `backup.json`  
     - Commit Message: Use expression `=n8n_backup_{{ $now.format('yyyy-MM-dd hh') }}`  
     - Binary Data: Enabled (upload file as binary)  
   - Set the node to **continue on error** to allow fallback logic.  
   - Add GitHub API credentials (OAuth2 or personal access token with repo access).

5. **Create If Node "If file not exits?"**  
   - Connect Edit a file error output → If file not exits?  
   - Configure condition:  
     - Check if JSON property `$json.error` equals `"The resource you are requesting could not be found"` (case sensitive)  

6. **Create Merge Node "Get Previous FIle back"**  
   - Connect:  
     - Main input 1: Move Binary Data (original backup content)  
     - Main input 2: From If node "true" output (file not found branch)  
   - Configure:  
     - Mode: Combine  
     - Join Mode: Enrich Input 1  
     - Fields to match: `data`  

7. **Create GitHub Node "Create a file"**  
   - Connect Get Previous FIle back → Create a file  
   - Configure similarly to "Edit a file" node but operation set to **Create**  
   - Same owner, repo, file path, commit message, and credentials.  

8. **Add Sticky Notes for Documentation**  
   - Create notes to explain backup interval, workflow purpose, and conditional logic as per original content.  

9. **Activate the Workflow**  
   - Save all nodes and activate the workflow.  
   - Test by running manually or waiting for the scheduled interval.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| ## Backup every 6 Hour (can be changed as per requirement)                                                                            | Backup interval setting note                                    |
| # 🚀 Automated n8n Workflow Backup to GitHub for schedule Interval ... (full detailed project description including setup and tips) | Detailed project overview and instructions (Sticky Note2)       |
| ## Check if File is present or not else it will create file a new file based on error                                                  | Explains the conditional logic to handle missing backup files  |
| Use GitHub branch protection rules for enhanced workflow security. Pair with Slack or Email notifications for backup alerts.         | Pro tips for improving workflow reliability and monitoring     |
| Ensure the n8n API key and GitHub credentials are securely stored and never exposed publicly.                                          | Security best practices                                         |
| Replace placeholder URLs and API keys with your own instance information before use.                                                  | Setup requirement                                               |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow. All data handled is legal and public. The workflow respects content policies and includes no illegal or offensive material.