Automated GitLab Backup of Workflows with Username Organization

https://n8nworkflows.xyz/workflows/automated-gitlab-backup-of-workflows-with-username-organization-5616


# Automated GitLab Backup of Workflows with Username Organization

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows into a GitLab repository. It organizes the backups by the username of the workflow creator, enabling structured archival and easy retrieval. The workflow supports both manual triggering and scheduled weekly execution.

The logic is divided into three main blocks:  
- **1.1 Backup Initiation:** Triggers the backup process either manually or on a weekly schedule.  
- **1.2 Workflow Retrieval & Metadata Preparation:** Fetches all existing n8n workflows and prepares backup metadata such as backup date and storage paths.  
- **1.3 Processing, Storage, and Notification:** Iterates over each workflow, formats it for GitLab, controls rate limits, checks for errors, updates or creates files in GitLab organized by creator username, logs results, and sends a notification email upon completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Backup Initiation

**Overview:**  
This block initiates the backup process either manually by user action or automatically on a scheduled weekly basis (Fridays at 18:00 UTC).

**Nodes Involved:**  
- Manual Backup Trigger  
- Scheduled Weekly Backup  
- Fetch N8N Workflows

**Node Details:**  

- **Manual Backup Trigger**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually on demand.  
  - Config: No parameters.  
  - Input: None (trigger node).  
  - Output: Connects to Fetch N8N Workflows.  
  - Edge Cases: None typical; manual triggers rely on user interaction.  

- **Scheduled Weekly Backup**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every week on Friday at 18:00 UTC.  
  - Config: Weekly interval, day = Friday (5), hour = 18.  
  - Input: None (trigger node).  
  - Output: Connects to Fetch N8N Workflows.  
  - Edge Cases: Timezone considerations; relies on correct n8n server time.  

- **Fetch N8N Workflows**  
  - Type: n8n API Node  
  - Role: Retrieves all existing workflows from the current n8n instance.  
  - Config: No filters applied; fetches all workflows.  
  - Credentials: Uses configured n8n API credentials.  
  - Input: Trigger from either manual or scheduled trigger.  
  - Output: Sends workflows data to Prepare Backup Metadata node.  
  - Edge Cases: API authentication failures, network issues, empty workflow list.  

---

#### 1.2 Workflow Retrieval & Metadata Preparation

**Overview:**  
Prepares metadata for the backup, including backup date and folder path. Then splits the workflows into batches for iterative processing.

**Nodes Involved:**  
- Prepare Backup Metadata  
- Process Each Workflow

**Node Details:**  

- **Prepare Backup Metadata**  
  - Type: Set Node  
  - Role: Assigns metadata fields such as backupDate (current UTC date) and backupPath (year/month folder) to the incoming workflows.  
  - Config: Uses expressions with `$now.setZone('UTC')` to format dates.  
  - Input: Workflows list from Fetch N8N Workflows.  
  - Output: Sends enriched workflow data to batch processor.  
  - Edge Cases: Date formatting errors if `$now` is unavailable (unlikely).  

- **Process Each Workflow**  
  - Type: SplitInBatches  
  - Role: Processes each workflow individually to avoid large bulk operations.  
  - Config: Default batch size (1 item per batch).  
  - Input: Workflows with metadata from Prepare Backup Metadata.  
  - Output: Splits the workflows for individual processing in subsequent nodes.  
  - Edge Cases: Large workflow sets may cause long execution times; no explicit batch size set (default).  

---

#### 1.3 Processing, Storage, and Notification

**Overview:**  
For each workflow, formats data for GitLab storage, controls API call rate limits, creates or updates files in GitLab under folders named by creator username, handles errors, logs results, and sends an email notification after all workflows are processed.

**Nodes Involved:**  
- Format Workflow for GitLab  
- Rate Limit Control  
- Create to GitLab Repository  
- Check Backup Status  
- Update Backup Summary  
- Log Backup Results  
- Send email

**Node Details:**  

- **Format Workflow for GitLab**  
  - Type: Set Node  
  - Role: Prepares a clean object containing essential workflow information for backup.  
  - Config:  
    - Extracts fields like id, name, nodes, connections, settings, tags, versionId, triggerCount, createdAt, updatedAt.  
    - Determines `creatorName` by inspecting workflow name for usernames ('vrushti', 'poojan', 'pragnesh', 'ajay'); defaults to 'general'.  
    - Sanitizes workflow name to generate safe fileName and filePath strings.  
  - Input: Individual workflow from Process Each Workflow.  
  - Output: Passes formatted workflow JSON and metadata to Rate Limit Control.  
  - Edge Cases: Workflow names without matching usernames go to 'general'; unexpected characters in names handled by regex replacement.  

- **Rate Limit Control**  
  - Type: Wait Node  
  - Role: Introduces a 2-second pause between GitLab API calls to avoid hitting rate limits.  
  - Config: Wait time set to 2 seconds.  
  - Input: Formatted workflow data.  
  - Output: Proceeds to create or update GitLab file.  
  - Edge Cases: Long total runtime if many workflows; no dynamic delay adjustment.  

- **Create to GitLab Repository**  
  - Type: GitLab Node  
  - Role: Attempts to create a new file in GitLab repository under user-named folder with workflow JSON content.  
  - Config:  
    - Owner: 'gitlab-user'  
    - Branch: 'main'  
    - Repository: 'n8n-backup'  
    - File path and content derived from formatted workflow.  
    - Commit message reflects addition and username.  
  - Credentials: Uses GitLab API credentials.  
  - Input: After rate limit wait.  
  - Output: Passes response to Check Backup Status.  
  - Edge Cases: If file exists, GitLab returns error (Bad request), triggering fallback to update.  

- **Check Backup Status**  
  - Type: If Node  
  - Role: Checks if the create operation failed due to file existing ("Bad request - please check your parameters").  
  - Config: Compares error message string.  
  - Input: Response from Create to GitLab Repository.  
  - Output:  
    - If error equals "Bad request - please check your parameters", calls Update Backup Summary (to update existing file).  
    - Else, loops back to Process Each Workflow (to continue).  
  - Edge Cases: Only handles specific error; other errors pass silently.  

- **Update Backup Summary**  
  - Type: GitLab Node  
  - Role: Updates existing workflow JSON file in GitLab repository.  
  - Config: Similar to Create node but uses 'edit' operation.  
  - Credentials: GitLab API credentials.  
  - Input: From Check Backup Status on error condition.  
  - Output: Loops back to Process Each Workflow to continue processing next.  
  - Edge Cases: Update failures not explicitly handled; continueOnFail is true in Create and Update nodes to avoid stopping workflow.  

- **Log Backup Results**  
  - Type: Set Node  
  - Role: Logs success or failure messages for each workflow processed.  
  - Config:  
    - If error present, logs failure with workflow fileName and error message.  
    - Else logs success with fileName and creatorName.  
  - Input: From Process Each Workflow.  
  - Output: Sends data to Send Email node.  
  - Edge Cases: Missing error messages default to 'Unknown error'.  

- **Send email**  
  - Type: Email Send Node  
  - Role: Sends notification email after processing all workflows.  
  - Config:  
    - Subject: "n8n workflows backup notification"  
    - Body: Confirmation message about successful backup completion.  
    - To: "abc.gmail.com" (example placeholder)  
    - From: "xyz@gmail.com" (example placeholder)  
  - Credentials: SMTP credentials configured.  
  - Input: From Log Backup Results node.  
  - Edge Cases: Email delivery failures not explicitly handled; single email sent after all workflows logged.  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                      | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|-------------------------|---------------------|-----------------------------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Manual Backup Trigger    | Manual Trigger      | Manual initiation of backup process                  | None                    | Fetch N8N Workflows      | Backup Initiation: Triggers manually or weekly to start the process                            |
| Scheduled Weekly Backup  | Schedule Trigger    | Scheduled weekly initiation (Friday 18:00 UTC)       | None                    | Fetch N8N Workflows      | Backup Initiation: Triggers manually or weekly to start the process                            |
| Fetch N8N Workflows      | n8n API             | Retrieves all workflows from n8n                      | Manual Backup Trigger, Scheduled Weekly Backup | Prepare Backup Metadata | Workflow Retrieval & Prep: Fetches workflows and prepares metadata                            |
| Prepare Backup Metadata  | Set                 | Adds backup date and path metadata                    | Fetch N8N Workflows      | Process Each Workflow    | Workflow Retrieval & Prep: Fetches workflows and prepares metadata                            |
| Process Each Workflow    | SplitInBatches      | Processes workflows individually                       | Prepare Backup Metadata, Check Backup Status, Update Backup Summary | Log Backup Results, Format Workflow for GitLab | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Format Workflow for GitLab | Set                 | Formats workflow data and organizes by creator username | Process Each Workflow    | Rate Limit Control       | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Rate Limit Control       | Wait                | Adds delay to control GitLab API rate limits          | Format Workflow for GitLab | Create to GitLab Repository | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Create to GitLab Repository | GitLab              | Creates new workflow file in GitLab repo               | Rate Limit Control       | Check Backup Status      | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Check Backup Status     | If                   | Checks if create failed due to existing file; triggers update | Create to GitLab Repository | Update Backup Summary (if exists), Process Each Workflow (else) | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Update Backup Summary    | GitLab              | Updates existing workflow file in GitLab              | Check Backup Status      | Process Each Workflow    | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Log Backup Results       | Set                 | Logs success or failure of each workflow backup       | Process Each Workflow    | Send email               | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Send email              | Email Send          | Sends notification email after backup completion      | Log Backup Results       | None                    | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |
| Sticky Note             | Sticky Note         | Documentation note                                     | None                    | None                    | Backup Initiation: Triggers manually or weekly to start the process                            |
| Sticky Note1            | Sticky Note         | Documentation note                                     | None                    | None                    | Workflow Retrieval & Prep: Fetches workflows and prepares metadata                            |
| Sticky Note2            | Sticky Note         | Documentation note                                     | None                    | None                    | Combined Processing, Storage, and Notification: Processes and formats workflows, saves to GitLab, logs, and notifies |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Backup Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed.  

2. **Create Scheduled Weekly Backup Node**  
   - Type: Schedule Trigger  
   - Configure rule: interval set to weekly, trigger at day = Friday (5), hour = 18 (UTC).  

3. **Create Fetch N8N Workflows Node**  
   - Type: n8n API Node  
   - Set operation to list workflows with no filters (fetch all).  
   - Add credentials for n8n API access.  
   - Connect outputs of Manual Backup Trigger and Scheduled Weekly Backup to this node.  

4. **Create Prepare Backup Metadata Node**  
   - Type: Set Node  
   - Add fields:  
     - `backupDate` = `{{$now.setZone('UTC').toFormat('yyyy-MM-dd')}}`  
     - `backupPath` = `backups/{{$now.setZone('UTC').toFormat('yyyy')}}/{{$now.setZone('UTC').toFormat('MM')}}`  
   - Connect Fetch N8N Workflows output to this node.  

5. **Create Process Each Workflow Node**  
   - Type: SplitInBatches  
   - Use default batch size (1).  
   - Connect Prepare Backup Metadata output to this node.  

6. **Create Format Workflow for GitLab Node**  
   - Type: Set Node  
   - Assign fields:  
     - `workflowData`: object with selected workflow properties (id, name, active, nodes, connections, settings, staticData, tags, versionId, triggerCount, createdAt, updatedAt).  
     - `creatorName`: expression to detect username from workflow name (check if name includes 'vrushti', 'poojan', 'pragnesh', 'ajay'; else 'general').  
     - `fileName`: sanitized workflow name with only alphanumeric, dash, underscore, replacing multiple underscores with one, trimmed of leading/trailing underscores, plus '.json'.  
     - `filePath`: path string structured as `n8n-workflows/{creatorName}/{fileName}`.  
   - Connect Process Each Workflow output to this node.  

7. **Create Rate Limit Control Node**  
   - Type: Wait Node  
   - Set wait time to 2 seconds.  
   - Connect Format Workflow for GitLab output to this node.  

8. **Create Create to GitLab Repository Node**  
   - Type: GitLab Node  
   - Configure:  
     - Owner: `gitlab-user`  
     - Branch: `main`  
     - Repository: `n8n-backup`  
     - FilePath: use expression `{{$json.filePath}}`  
     - FileContent: JSON.stringify(`$json.workflowData`, pretty print)  
     - Commit Message: `Add {{$json.fileName}} - {{$json.creatorName}} folder via n8n automation`  
   - Add GitLab API credentials.  
   - Set continueOnFail = true.  
   - Connect Rate Limit Control output to this node.  

9. **Create Check Backup Status Node**  
   - Type: If Node  
   - Condition: Check if `$json.error` equals string "Bad request - please check your parameters".  
   - Connect Create to GitLab Repository output to this node.  

10. **Create Update Backup Summary Node**  
    - Type: GitLab Node  
    - Configure same as Create node but operation = `edit`.  
    - Use filePath and commit message similar to Create node but commit message: `Update {{$json.fileName}} in {{$json.creatorName}} folder via n8n automation`.  
    - Add GitLab API credentials.  
    - Set continueOnFail = true.  
    - Connect If Node’s “true” output (error condition) to this node.  

11. **Connect Update Backup Summary output back to Process Each Workflow**  
    - Enables processing of next workflow after update.  

12. **Connect If Node’s “false” output directly to Process Each Workflow**  
    - Continues if create succeeded without error.  

13. **Create Log Backup Results Node**  
    - Type: Set Node  
    - Assign fields:  
      - `errorLog`: `"Failed to process {{$json.fileName}}: {{$json.error?.message || 'Unknown error'}}"`  
      - `successLog`: `"Successfully processed {{$json.fileName}} for {{$json.creatorName}}"`  
    - Connect Process Each Workflow first output (batch) to this node.  

14. **Create Send Email Node**  
    - Type: Email Send  
    - Configure:  
      - Subject: "n8n workflows backup notification"  
      - Text: Static message confirming successful backup completion.  
      - To Email: Set actual recipient email (replace placeholder).  
      - From Email: Set actual sender email (replace placeholder).  
      - Credentials: SMTP credentials configured.  
    - Connect Log Backup Results output to this node.  

15. **Optional: Add Sticky Notes**  
    - Add notes near logical blocks for documentation clarity:
      - Backup Initiation (manual and scheduled triggers)  
      - Workflow Retrieval & Preparation  
      - Processing, Storage, and Notification  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow organizes backups into GitLab folders based on detected usernames in workflow names. | Helps maintain structured backups by user ownership.                                          |
| Rate limiting set to 2 seconds between GitLab API calls to avoid hitting API limits.           | Important for stable execution when backing up many workflows.                                |
| Email notification sent to confirm backup success; replace placeholder emails with real ones. | Ensure SMTP credentials and email addresses are properly configured before deployment.        |
| Uses n8n API credentials to fetch workflows; ensure proper permissions are granted.            | Missing permissions may cause API call failures.                                              |
| GitLab nodes use 'continueOnFail' to ensure workflow continues despite individual file errors. | Prevents entire backup from stopping due to one file conflict or error.                       |
| Usernames in workflow names must be lowercase and included explicitly for correct foldering.  | Adjust username detection logic as needed for your organization’s naming conventions.         |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.