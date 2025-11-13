Automated Workflow Backups to Google Drive with Smart Cleanup

https://n8nworkflows.xyz/workflows/automated-workflow-backups-to-google-drive-with-smart-cleanup-7512


# Automated Workflow Backups to Google Drive with Smart Cleanup

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows to Google Drive and performs automatic cleanup of old backups to manage storage efficiently. It is designed for daily execution (or manual triggering) to ensure regular snapshots of workflows are securely saved while keeping Google Drive organized.

The workflow consists of two main logical blocks:

- **1.1 Backup Block:**  
  Creates a new dated folder in Google Drive, fetches all n8n workflows via API, converts each workflow into a JSON file, and uploads them into the newly created folder.

- **1.2 Cleanup Block:**  
  Searches for existing backup folders in Google Drive, sorts them by creation date, and deletes all but the most recent specified number of backup folders to free up space.

---

### 2. Block-by-Block Analysis

#### 2.1 Backup Block

**Overview:**  
This block creates a new Google Drive folder named with the current date, retrieves all existing n8n workflows, converts each workflow into a JSON file, and uploads these files to the newly created folder.

**Nodes Involved:**  
- CONFIG - Set your variables here  
- Create new backup folder  
- Get all n8n workflows  
- Loop over each workflow  
- Convert workflow to file  
- Upload workflow to Google Drive

**Node Details:**

- **CONFIG - Set your variables here**  
  - Type: Set  
  - Role: Holds configuration variables for the workflow, including the parent Google Drive folder ID and the number of backups to keep.  
  - Key variables:  
    - `parentFolderId`: Google Drive folder ID where backups are stored.  
    - `backupsToKeep`: Number of recent backups to retain (e.g., 30).  
  - Inputs: Manual or schedule trigger → configuration node.  
  - Outputs: Passes config variables downstream.  
  - Edge cases: If incorrect or missing `parentFolderId`, Google Drive operations will fail with authorization or not-found errors.

- **Create new backup folder**  
  - Type: Google Drive (folder creation)  
  - Role: Creates a new folder inside the configured parent folder with a name like "Workflows backup - YYYY-MM-DD".  
  - Configuration:  
    - Folder name dynamically uses the current date (`$now.format('yyyy-MM-dd')`).  
    - Parent folder ID from config node.  
    - Drive ID set to "My Drive".  
  - Inputs: Receives from config node.  
  - Outputs: New folder metadata, including the folder ID for uploads.  
  - Edge cases: Failure if the parent folder ID is invalid or API quota is exceeded.

- **Get all n8n workflows**  
  - Type: n8n API node  
  - Role: Requests a list of all workflows from the current n8n instance.  
  - Configuration: No filters; fetches all workflows.  
  - Credential: Requires valid n8n API credentials.  
  - Inputs: From the folder creation node.  
  - Outputs: List of workflows metadata.  
  - Edge cases: API authentication failure, API rate limit, or connectivity issues.

- **Loop over each workflow**  
  - Type: SplitInBatches  
  - Role: Iterates through each workflow item to process them individually.  
  - Configuration: Default batch size (usually 1).  
  - Inputs: List of workflows from the API node.  
  - Outputs: Sends one workflow at a time downstream.  
  - Edge cases: Large workflow lists may slow execution; batch size can be adjusted.

- **Convert workflow to file**  
  - Type: ConvertToFile  
  - Role: Converts the workflow JSON data to a `.json` file format suitable for upload.  
  - Configuration:  
    - Format output to JSON.  
    - Filename derived from workflow name with `.json` extension.  
  - Inputs: Single workflow JSON from loop.  
  - Outputs: File object ready for upload.  
  - Edge cases: Workflow JSON malformed; file naming collisions unlikely due to unique workflow names.

- **Upload workflow to Google Drive**  
  - Type: Google Drive (file upload)  
  - Role: Uploads the JSON workflow file into the newly created dated folder.  
  - Configuration:  
    - File name matches workflow name + `.json`.  
    - Upload location is the folder ID from "Create new backup folder".  
    - Drive ID is "My Drive".  
  - Credential: Google Drive OAuth2 required.  
  - Inputs: File from convert node.  
  - Outputs: Confirmation of upload success.  
  - Edge cases: Upload failure due to API limits, folder ID errors, or connectivity issues.

---

#### 2.2 Cleanup Block

**Overview:**  
This block manages storage by searching Google Drive for existing backup folders, sorting them by creation date, and deleting all but the configured number of most recent backups.

**Nodes Involved:**  
- Start cleanup (run once)  
- Search files and folders  
- Sort and isolate old folders  
- Loop over folders to delete  
- Delete old folder

**Node Details:**

- **Start cleanup (run once)**  
  - Type: Set  
  - Role: Acts as an initial trigger to start the cleanup process; configured to execute only once per run.  
  - Inputs: From the loop over workflows node (after backup completion).  
  - Outputs: Initiates search for backup folders.  
  - Edge cases: Ensures cleanup doesn’t run multiple times per backup cycle.

- **Search files and folders**  
  - Type: Google Drive (search)  
  - Role: Searches the configured Google Drive parent folder for folders matching the name prefix "Workflows backup - ".  
  - Configuration:  
    - Folder ID from config node.  
    - Query string filters only folders starting with "Workflows backup - ".  
    - Returns all matching items.  
  - Credential: Google Drive OAuth2 required.  
  - Inputs: From "Start cleanup".  
  - Outputs: List of backup folders.  
  - Edge cases: Search failure, permission errors, or empty folder list.

- **Sort and isolate old folders**  
  - Type: Code (JavaScript)  
  - Role: Sorts the list of backup folders from newest to oldest and slices off the number of backups to keep, isolating folders to delete.  
  - Configuration:  
    - Reads `backupsToKeep` from config node.  
    - Sorts by `createdTime` descending.  
    - Returns array of folders exceeding the keep count.  
  - Inputs: Search result from Google Drive.  
  - Outputs: List of folders to delete.  
  - Edge cases: No folders found; empty deletion list.

- **Loop over folders to delete**  
  - Type: SplitInBatches  
  - Role: Iterates through each folder identified for deletion.  
  - Configuration: Default batch size.  
  - Inputs: List from code node.  
  - Outputs: Each folder metadata passed individually for deletion.  
  - Edge cases: Large deletion list may require batch adjustment.

- **Delete old folder**  
  - Type: Google Drive (delete folder)  
  - Role: Deletes the specified folder from Google Drive.  
  - Configuration:  
    - Uses folder ID from each item passed by the loop.  
  - Credential: Google Drive OAuth2 required.  
  - Inputs: Folder metadata from loop.  
  - Outputs: Confirmation of deletion.  
  - Edge cases: Folder may be locked, already deleted, or permission denied.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                                  | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                                       |
|--------------------------|-------------------------|-------------------------------------------------|-------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger        | Triggers workflow automatically at 3 AM daily  | —                             | CONFIG - Set your variables here | Backup n8n workflows to Google Drive with automatic cleanup (summary and instructions)                                           |
| Manual trigger           | Manual Trigger          | Manual workflow start                           | —                             | CONFIG - Set your variables here | Backup n8n workflows to Google Drive with automatic cleanup (summary and instructions)                                           |
| CONFIG - Set your variables here | Set                     | Holds configuration variables                   | Schedule Trigger, Manual trigger | Create new backup folder          | How to find your Google Drive Folder ID (instructions and example)                                                              |
| Create new backup folder | Google Drive (folder)    | Creates dated backup folder                      | CONFIG - Set your variables here | Get all n8n workflows             | Backup Logic: Fetches all workflows and uploads them to a new daily folder                                                       |
| Get all n8n workflows    | n8n API                 | Retrieves all workflows from n8n instance       | Create new backup folder        | Loop over each workflow           | Backup Logic: Fetches all workflows and uploads them to a new daily folder                                                       |
| Loop over each workflow  | SplitInBatches          | Iterates through each workflow                   | Get all n8n workflows           | Start cleanup (run once), Convert workflow to file | Backup Logic: Fetches all workflows and uploads them to a new daily folder                                                       |
| Start cleanup (run once) | Set                     | Initializes cleanup sequence                      | Loop over each workflow         | Search files and folders          | Cleanup Logic: Finds and deletes old backup folders                                                                              |
| Search files and folders | Google Drive (search)    | Finds existing backup folders in Google Drive   | Start cleanup (run once)        | Sort and isolate old folders      | Cleanup Logic: Finds and deletes old backup folders                                                                              |
| Sort and isolate old folders | Code                    | Sorts backup folders and selects old ones for deletion | Search files and folders        | Loop over folders to delete       | Cleanup Logic: Finds and deletes old backup folders                                                                              |
| Loop over folders to delete | SplitInBatches          | Iterates over folders selected for deletion     | Sort and isolate old folders    | Delete old folder                 | Cleanup Logic: Finds and deletes old backup folders                                                                              |
| Delete old folder        | Google Drive (delete)    | Deletes old backup folders                        | Loop over folders to delete     | Loop over folders to delete       | Cleanup Logic: Finds and deletes old backup folders                                                                              |
| Convert workflow to file | ConvertToFile           | Converts individual workflow JSON to file       | Loop over each workflow         | Upload workflow to Google Drive   | Backup Logic: Fetches all workflows and uploads them to a new daily folder                                                       |
| Upload workflow to Google Drive | Google Drive (upload)    | Uploads workflow JSON file to Google Drive       | Convert workflow to file        | Loop over each workflow           | Backup Logic: Fetches all workflows and uploads them to a new daily folder                                                       |
| Sticky Note              | Sticky Note             | Instructional note on finding Google Drive Folder ID | —                             | —                               | How to find your Google Drive Folder ID (instructions and example)                                                              |
| Sticky Note1             | Sticky Note             | Explains cleanup logic block                      | —                             | —                               | Cleanup Logic: Finds and deletes old backup folders                                                                              |
| Sticky Note2             | Sticky Note             | Explains backup logic block                       | —                             | —                               | Backup Logic: Fetches all workflows and uploads them to a new daily folder                                                       |
| Sticky Note3             | Sticky Note             | Workflow overview and setup instructions         | —                             | —                               | Backup n8n workflows to Google Drive with automatic cleanup (summary and instructions)                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node:  
     - Set to run daily at 3:00 AM.  
   - Add a **Manual Trigger** node for manual runs.

2. **Create Configuration Node:**  
   - Add a **Set** node named `CONFIG - Set your variables here`.  
   - Add two variables:  
     - `parentFolderId` (string): Paste your Google Drive folder ID where backups will be stored.  
     - `backupsToKeep` (number): Set how many backups to keep, e.g., 30.

3. **Create New Backup Folder Node:**  
   - Add a **Google Drive** node named `Create new backup folder`.  
   - Operation: Create Folder.  
   - Folder name: Use expression `Workflows backup - {{ $now.format('yyyy-MM-dd') }}`.  
   - Parent Folder ID: Use expression from config node `parentFolderId`.  
   - Drive ID: Select "My Drive".  
   - Connect input from the `CONFIG` node.

4. **Get All Workflows Node:**  
   - Add an **n8n** node named `Get all n8n workflows`.  
   - No filters applied (to get all workflows).  
   - Connect input from `Create new backup folder`.  
   - Set credentials with valid n8n API credentials.

5. **Loop Over Each Workflow:**  
   - Add a **SplitInBatches** node named `Loop over each workflow`.  
   - Connect input from `Get all n8n workflows`.

6. **Convert Workflow to File:**  
   - Add a **ConvertToFile** node named `Convert workflow to file`.  
   - Operation: Convert JSON to file.  
   - File name: Expression `{{$json.name + ".json"}}`.  
   - Connect input from `Loop over each workflow`.

7. **Upload Workflow to Google Drive:**  
   - Add a **Google Drive** node named `Upload workflow to Google Drive`.  
   - Operation: Upload file.  
   - File name: Use expression `{{$json.name + ".json"}}`.  
   - Drive ID: Select "My Drive".  
   - Folder ID: Use folder ID from `Create new backup folder` node.  
   - Connect input from `Convert workflow to file`.  
   - Use Google Drive OAuth2 credentials.

8. **Backup Completion Trigger Cleanup:**  
   - Connect a second output from `Loop over each workflow` to a **Set** node named `Start cleanup (run once)`.  
   - Enable "Execute once" option to ensure it runs a single time per workflow execution.

9. **Search Existing Backup Folders:**  
   - Add a **Google Drive** node named `Search files and folders`.  
   - Operation: Search files/folders.  
   - Query: `"Workflows backup - "` (to find backup folders).  
   - Folder ID: From config node `parentFolderId`.  
   - Drive ID: "My Drive".  
   - Connect input from `Start cleanup (run once)`.

10. **Sort and Isolate Old Folders:**  
    - Add a **Code** node named `Sort and isolate old folders`.  
    - JavaScript code:  
      ```js
      const backupsToKeep = $json.backupsToKeep;
      const folders = $input.all();
      const sortedFolders = folders.map(item => item.json);
      sortedFolders.sort((a, b) => new Date(b.createdTime) - new Date(a.createdTime));
      const foldersToDelete = sortedFolders.slice(backupsToKeep);
      return foldersToDelete;
      ```  
    - Connect input from `Search files and folders`.

11. **Loop Over Folders to Delete:**  
    - Add a **SplitInBatches** node named `Loop over folders to delete`.  
    - Connect input from `Sort and isolate old folders`.

12. **Delete Old Folder:**  
    - Add a **Google Drive** node named `Delete old folder`.  
    - Operation: Delete folder.  
    - Folder ID: Use expression `{{$json.id}}` from input.  
    - Drive ID: "My Drive".  
    - Connect input from `Loop over folders to delete`.  
    - Use Google Drive OAuth2 credentials.

13. **Connect Delete Confirmation:**  
    - Connect the output of `Delete old folder` back to the second output of `Loop over folders to delete` to continue processing batches until all folders are deleted.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| How to find your Google Drive Folder ID: Navigate to your desired folder in Google Drive, then copy the folder ID from the URL (the long string after `/folders/`). Paste this ID into the `parentFolderId` variable in the config node. Example URL: `https://drive.google.com/drive/folders/1Fs3gg7pSVhbvODJQLcgAfP-V0rCGC_Oc` → Folder ID: `1Fs3gg7pSVhbvODJQLcgAfP-V0rCGC_Oc`.                                                                                   | [Google Drive Folder ID Instructions](https://drive.google.com)                                                          |
| This workflow automates backup and cleanup of n8n workflows in Google Drive. It runs daily at 3 AM by default but can also be triggered manually. It keeps your backup storage organized by deleting old backup folders beyond the configured retention count.                                                                                                                                                                            | Workflow overview and general instructions (Sticky Note)                                                                 |
| Ensure valid credentials are configured for both n8n API access and Google Drive OAuth2 before activating the workflow to avoid failures due to authentication or authorization errors.                                                                                                                                                                                                                                               | Credential setup instructions                                                                                             |
| Batch processing is used for both workflows iteration and folder deletion to handle large datasets efficiently and avoid timeouts or API rate limits. Batch sizes can be adjusted if needed.                                                                                                                                                                                                                                            | Performance and scalability considerations                                                                                 |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected materials. All handled data is legal and public.