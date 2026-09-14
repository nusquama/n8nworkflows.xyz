Archive Discord attachments to Google Drive and log them in Google Sheets

https://n8nworkflows.xyz/workflows/archive-discord-attachments-to-google-drive-and-log-them-in-google-sheets-17615


# Archive Discord attachments to Google Drive and log them in Google Sheets

### 1. Workflow Overview

This workflow automates the process of backing up Discord attachments to Google Drive and recording detailed metadata in Google Sheets. It prevents files from becoming inaccessible due to the expiration of Discord’s Content Delivery Network (CDN) signed URLs. 

The execution model follows a sequential, deterministic pipeline divided into the following logical blocks:

- **1.1 Input Reception & Configuration Initialization:** Triggers execution on a fixed daily schedule, initializes core configuration variables (Discord server, target channels, Google Drive/Sheet pointers), and reads the existing archive log to establish an execution watermark.
- **1.2 Channel Iteration & Message Ingestion:** Processes configuration data into channel-specific inputs, fetches recent messages via the Discord API, filters out older or text-only messages, and screens attachments against size thresholds.
- **1.3 Drive Organization & Binary Processing:** Dynamically generates a dated Google Drive destination folder, plans individual file uploads with sanitized naming conventions, downloads attachment binaries via signed HTTP URLs, and persists them to Google Drive.
- **1.4 Logging & Audit Trail:** Appends structured metadata (author, timestamps, file properties, and Google Drive links) back to the Google Sheets archive log for downstream tracking and reference.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Input Reception & Configuration Initialization

#### Overview
This block initiates the workflow daily, defines global operational parameters (such as target guilds, channel arrays, directory identifiers, and file size ceilings), reads historical state from Google Sheets, and computes a synchronization watermark.

#### Nodes Involved
- `Run Daily Archive Sweep`
- `Set Archive Settings`
- `Read Archive Log`
- `Build Channel List`

---

#### Node Details

##### Run Daily Archive Sweep
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Starts the execution pipeline based on a time-based interval.
- **Configuration Choices:** Configured to execute daily at hour 3 (03:00 UTC).
- **Key Expressions or Variables:** None (time-based configuration).
- **Input / Output Connections:** Input: None (Root trigger) | Output: `Set Archive Settings` (main path).
- **Version-Specific Requirements:** Version 1.3.
- **Edge Cases / Failure Types:** Missed schedules if the n8n instance is offline at the exact trigger hour. Ensure system time and timezone match expectations.
- **Sub-Workflow Reference:** None.

##### Set Archive Settings
- **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation / Parameter Node). Establishes global static settings and limits for the runtime session.
- **Configuration Choices:** Sets explicit string and numeric parameters via assignment fields:
  - `guildId`: Discord server identifier.
  - `channelIds`: Comma-separated Discord channel identifiers.
  - `driveParentFolderId`: Destination parent directory in Google Drive.
  - `logSheetUrl`: Target tracking document URL.
  - `logTabName`: Target worksheet tab (`Archive Log`).
  - `maxFileMb`: Maximum allowed file size in megabytes (`25`).
  - `firstRunLookbackDays`: Fallback lookback window for empty logs (`1`).
- **Key Expressions or Variables:** Static string literals containing placeholder values that must be replaced before production deployment.
- **Input / Output Connections:** Input: `Run Daily Archive Sweep` | Output: `Read Archive Log`.
- **Version-Specific Requirements:** Version 3.4.
- **Edge Cases / Failure Types:** Failure to replace placeholder strings will cause downstream API rejections (invalid IDs or URLs).
- **Sub-Workflow Reference:** None.

##### Read Archive Log
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Integration Node). Retrieves existing tracking data from the target spreadsheet to identify previously processed files.
- **Configuration Choices:** 
  - Operation: Read (implicit via document and sheet resolution).
  - Document ID: Evaluates dynamically from settings (`logSheetUrl`).
  - Sheet Name: Evaluates dynamically from settings (`logTabName`).
  - Options: `alwaysOutputData` is enabled to ensure the workflow proceeds even if the sheet contains no prior rows.
- **Key Expressions or Variables:** 
  - Document ID: `={{ $("Set Archive Settings").first().json.logSheetUrl }}`
  - Sheet Name: `={{ $("Set Archive Settings").first().json.logTabName }}`
- **Input / Output Connections:** Input: `Set Archive Settings` | Output: `Build Channel List`.
- **Version-Specific Requirements:** Version 4.7. Requires valid Google Sheets OAuth2 credentials.
- **Edge Cases / Failure Types:** Authentication token expiry, incorrect spreadsheet URL, or missing sheet headers will throw an error or return malformed data.
- **Sub-Workflow Reference:** None.

##### Build Channel List
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Computes the execution watermark from historical log data and splits configured channels into individual workflow execution items.
- **Configuration Choices:** Custom JavaScript processing array transformation, date parsing, and validation checks.
- **Key Expressions or Variables:** Accesses upstream settings via `$('Set Archive Settings').first().json` and reads incoming rows via `$input.all()`.
- **Input / Output Connections:** Input: `Read Archive Log` | Output: `Get Recent Messages`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Failure Types:** 
  - Throws an explicit error if `guildId` is empty.
  - Throws an explicit error if `channelIds` resolves to an empty array.
  - Malformed date strings in the `Posted At` column are filtered out safely using `Number.isFinite(ms)`.
- **Sub-Workflow Reference:** None.

---

### Block 1.2: Channel Iteration & Message Ingestion

#### Overview
This block iterates through each target Discord channel, fetches recent message payloads, filters out already-archived messages via the computed watermark, and screens attachments against size constraints.

#### Nodes Involved
- `Get Recent Messages`
- `Extract New Attachments`
- `Skip Oversized Attachments`

---

#### Node Details

##### Get Recent Messages
- **Type and Technical Role:** `n8n-nodes-base.discord` (Integration Node). Communicates with the Discord API to fetch message history for specified channels.
- **Configuration Choices:**
  - Resource: `message`
  - Operation: `getAll`
  - Guild ID: `={{ $json.guildId }}`
  - Channel ID: `={{ $json.channelId }}`
  - Options: `simplify: false` (vital to preserve the raw nested attachment arrays).
  - Error Handling: `onError: continueRegularOutput` (prevents a single channel permission failure from halting the entire batch).
- **Key Expressions or Variables:** 
  - `={{ $json.guildId }}`
  - `={{ $json.channelId }}`
- **Input / Output Connections:** Input: `Build Channel List` | Output: `Extract New Attachments`.
- **Version-Specific Requirements:** Version 2. Requires Discord Bot credentials with `MESSAGE CONTENT` privileged intent and channel read permissions.
- **Edge Cases / Failure Types:** Rate-limiting (HTTP 429), missing bot permissions (HTTP 403), or invalid channel IDs. Handled gracefully via `continueRegularOutput`.
- **Sub-Workflow Reference:** None.

##### Extract New Attachments
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Filters raw Discord messages against the timestamp watermark and extracts valid file attachments.
- **Configuration Choices:** Custom JavaScript processing that strips duplicate attachment IDs, computes file sizes in KB, and formats message jump URLs.
- **Key Expressions or Variables:** References plan data via `$('Build Channel List').first().json` and processes message items using `$input.all()`.
- **Input / Output Connections:** Input: `Get Recent Messages` | Output: `Skip Oversized Attachments`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Failure Types:** Messages without attachments or messages older than the watermark are skipped. Results are sorted chronologically by timestamp.
- **Sub-Workflow Reference:** None.

##### Skip Oversized Attachments
- **Type and Technical Role:** `n8n-nodes-base.filter` (Logic / Control Flow Node). Drops attachments that exceed the user-defined size limitation before download execution.
- **Configuration Choices:**
  - Condition: Less than or equal to (`lte`)
  - Left Value: `={{ $json.fileSizeBytes }}`
  - Right Value: `={{ $("Set Archive Settings").first().json.maxFileMb * 1024 * 1024 }}`
- **Key Expressions or Variables:** Evaluates dynamic size caps from the configuration node.
- **Input / Output Connections:** Input: `Extract New Attachments` | Output: `Create Dated Drive Folder`.
- **Version-Specific Requirements:** Version 2.3.
- **Edge Cases / Failure Types:** Items exceeding the byte threshold are filtered out of the active data stream.
- **Sub-Workflow Reference:** None.

---

### Block 1.3: Drive Organization & Binary Processing

#### Overview
This block creates a dated directory structure in Google Drive, plans unique file naming conventions, downloads attachment binaries from ephemeral Discord CDN URLs, and uploads them to Google Drive.

#### Nodes Involved
- `Create Dated Drive Folder`
- `Plan Uploads Into Folder`
- `Download Attachment`
- `Upload To Drive Folder`

---

#### Node Details

##### Create Dated Drive Folder
- **Type and Technical Role:** `n8n-nodes-base.googleDrive` (Integration Node). Creates a new directory in Google Drive corresponding to the execution date.
- **Configuration Choices:**
  - Resource: `folder`
  - Operation: Create (implicit)
  - Name: `={{ $("Build Channel List").first().json.runDate }}`
  - Folder ID: `={{ $("Set Archive Settings").first().json.driveParentFolderId }}`
  - Execute Once: Enabled (`true`) to prevent duplicate folder creation across batched items.
- **Key Expressions or Variables:** 
  - Folder Name: `={{ $("Build Channel List").first().json.runDate }}`
  - Parent ID: `={{ $("Set Archive Settings").first().json.driveParentFolderId }}`
- **Input / Output Connections:** Input: `Skip Oversized Attachments` | Output: `Plan Uploads Into Folder`.
- **Version-Specific Requirements:** Version 3. Requires Google Drive OAuth2 credentials.
- **Edge Cases / Failure Types:** Invalid parent folder ID or insufficient write permissions will abort directory creation.
- **Sub-Workflow Reference:** None.

##### Plan Uploads Into Folder
- **Type and Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node). Sanitizes incoming filenames and injects Google Drive destination identifiers into the item schema.
- **Configuration Choices:** Custom JavaScript replacing unsafe path characters and special symbols with underscores.
- **Key Expressions or Variables:** References folder metadata via `$('Create Dated Drive Folder').first().json`.
- **Input / Output Connections:** Input: `Create Dated Drive Folder` | Output: `Download Attachment`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Failure Types:** Throws an error if the directory creation step failed to return a valid folder ID.
- **Sub-Workflow Reference:** None.

##### Download Attachment
- **Type and Technical Role:** `n8n-nodes-base.httpRequest` (Integration / Utility Node). Fetches the binary content of the file from Discord's ephemeral signed CDN URL.
- **Configuration Choices:**
  - URL: `={{ $json.downloadUrl }}`
  - Options:
    - Timeout: `60000` ms (60 seconds)
    - Response Format: `file` (Binary output)
    - Batching: Batch size configured to `5` concurrent requests.
- **Key Expressions or Variables:** `={{ $json.downloadUrl }}`
- **Input / Output Connections:** Input: `Plan Uploads Into Folder` | Output: `Upload To Drive Folder`.
- **Version-Specific Requirements:** Version 4.4.
- **Edge Cases / Failure Types:** 
  - Ephemeral CDN links returning HTTP 404 if execution is delayed.
  - Network timeouts on large binary payloads.
- **Sub-Workflow Reference:** None.

##### Upload To Drive Folder
- **Type and Technical Role:** `n8n-nodes-base.googleDrive` (Integration Node). Uploads the downloaded binary file into the previously created dated Google Drive directory.
- **Configuration Choices:**
  - Resource: File (implicit via binary upload)
  - Operation: Upload / Create
  - Name: `={{ $("Plan Uploads Into Folder").item.json.driveFileName }}`
  - Folder ID: `={{ $("Create Dated Drive Folder").first().json.id }}`
- **Key Expressions or Variables:** 
  - File Name: `={{ $("Plan Uploads Into Folder").item.json.driveFileName }}`
  - Folder ID: `={{ $("Create Dated Drive Folder").first().json.id }}`
- **Input / Output Connections:** Input: `Download Attachment` | Output: `Append To Archive Log`.
- **Version-Specific Requirements:** Version 3. Requires Google Drive OAuth2 credentials.
- **Edge Cases / Failure Types:** Drive storage quota limits or network interruptions during multipart binary transfer.
- **Sub-Workflow Reference:** None.

---

### Block 1.4: Logging & Audit Trail

#### Overview
This block records detailed transaction metadata and links into the Google Sheets audit log to maintain system state and enable incremental watermarking on subsequent runs.

#### Nodes Involved
- `Append To Archive Log`

---

#### Node Details

##### Append To Archive Log
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Integration Node). Appends a new audit row to the Google Sheets tracking tab.
- **Configuration Choices:**
  - Operation: `append`
  - Mapping Mode: `defineBelow`
  - Document ID: `={{ $("Set Archive Settings").first().json.logSheetUrl }}`
  - Sheet Name: `={{ $("Set Archive Settings").first().json.logTabName }}`
  - Column Mappings:
    - `File Name`: `={{ $("Plan Uploads Into Folder").item.json.fileName }}`
    - `Posted At`: `={{ $("Plan Uploads Into Folder").item.json.postedAt }}`
    - `Posted By`: `={{ $("Plan Uploads Into Folder").item.json.postedBy }}`
    - `Channel ID`: `={{ $("Plan Uploads Into Folder").item.json.channelId }}`
    - `Drive Link`: `={{ $json.webViewLink || "https://drive.google.com/file/d/" + $json.id + "/view" }}`
    - `Archived At`: `={{ $now.toISO() }}`
    - `Content Type`: `={{ $("Plan Uploads Into Folder").item.json.contentType }}`
    - `File Size KB`: `={{ $("Plan Uploads Into Folder").item.json.fileSizeKb }}`
    - `Message Link`: `={{ $("Plan Uploads Into Folder").item.json.messageLink }}`
    - `Drive File ID`: `={{ $json.id }}`
- **Key Expressions or Variables:** Utilizes item-scoped node references (`$().item.json`) to map contextual payload data alongside Drive API output properties (`$json.id`, `$json.webViewLink`).
- **Input / Output Connections:** Input: `Upload To Drive Folder` | Output: None (Terminal node).
- **Version-Specific Requirements:** Version 4.7. Requires Google Sheets OAuth2 credentials.
- **Edge Cases / Failure Types:** Column header mismatch between the node mapping and the physical spreadsheet will result in misaligned or rejected data entries.
- **Sub-Workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run Daily Archive Sweep` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow daily at 03:00 UTC. | None | `Set Archive Settings` | [Archive Discord attachments to Google Drive with a Google Sheets log] |
| `Set Archive Settings` | `n8n-nodes-base.set` | Initializes global configuration parameters and limits. | `Run Daily Archive Sweep` | `Read Archive Log` | Configure and find the watermark |
| `Read Archive Log` | `n8n-nodes-base.googleSheets` | Fetches existing log entries from Google Sheets. | `Set Archive Settings` | `Build Channel List` | Configure and find the watermark |
| `Build Channel List` | `n8n-nodes-base.code` | Computes the execution watermark and maps channels. | `Read Archive Log` | `Get Recent Messages` | Configure and find the watermark |
| `Get Recent Messages` | `n8n-nodes-base.discord` | Fetches message history from Discord channels. | `Build Channel List` | `Extract New Attachments` | Read the named channels |
| `Extract New Attachments` | `n8n-nodes-base.code` | Filters messages by timestamp and extracts file metadata. | `Get Recent Messages` | `Skip Oversized Attachments` | Read the named channels |
| `Skip Oversized Attachments` | `n8n-nodes-base.filter` | Drops attachments exceeding the configured size limit. | `Extract New Attachments` | `Create Dated Drive Folder` | Cap size and make the folder |
| `Create Dated Drive Folder` | `n8n-nodes-base.googleDrive` | Creates a dated destination folder in Google Drive. | `Skip Oversized Attachments` | `Plan Uploads Into Folder` | Cap size and make the folder |
| `Plan Uploads Into Folder` | `n8n-nodes-base.code` | Sanitizes filenames and assigns target folder IDs. | `Create Dated Drive Folder` | `Download Attachment` | Cap size and make the folder |
| `Download Attachment` | `n8n-nodes-base.httpRequest` | Downloads attachment binaries via signed URLs. | `Plan Uploads Into Folder` | `Upload To Drive Folder` | Download, upload and log |
| `Upload To Drive Folder` | `n8n-nodes-base.googleDrive` | Uploads binary files to the dated Google Drive folder. | `Download Attachment` | `Append To Archive Log` | Download, upload and log |
| `Append To Archive Log` | `n8n-nodes-base.googleSheets` | Appends transaction records to the tracking spreadsheet. | `Upload To Drive Folder` | None | Download, upload and log |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow in an n8n canvas manually:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Run Daily Archive Sweep`.
   - Set interval rule to trigger daily at hour `3`.

2. **Configure Global Settings:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Set Archive Settings`.
   - Connect `Run Daily Archive Sweep` to `Set Archive Settings`.
   - Add the following string and number assignments:
     - `guildId` (String): Your Discord Guild/Server ID.
     - `channelIds` (String): Comma-separated Discord Channel IDs.
     - `driveParentFolderId` (String): Destination Google Drive Folder ID.
     - `logSheetUrl` (String): Full URL of the Google Sheet log.
     - `logTabName` (String): `Archive Log`
     - `maxFileMb` (Number): `25`
     - `firstRunLookbackDays` (Number): `1`

3. **Set Up Historical Log Reading:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Read Archive Log`.
   - Connect `Set Archive Settings` to `Read Archive Log`.
   - Configure credentials (Google Sheets OAuth2).
   - Set **Document ID** to expression: `={{ $("Set Archive Settings").first().json.logSheetUrl }}`
   - Set **Sheet Name** to expression: `={{ $("Set Archive Settings").first().json.logTabName }}`
   - Enable **Always Output Data** in options.

4. **Build Channel Processing Logic:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Build Channel List`.
   - Connect `Read Archive Log` to `Build Channel List`.
   - Paste JavaScript snippet to parse log timestamps, compute the watermark (`watermarkMs`), validate guild/channel IDs, and map each channel into a distinct execution item.

5. **Integrate Discord Message Retrieval:**
   - Add a **Discord** node (`n8n-nodes-base.discord`) named `Get Recent Messages`.
   - Connect `Build Channel List` to `Get Recent Messages`.
   - Configure credentials (Discord Bot API).
   - Set **Resource** to `Message` and **Operation** to `Get Many` (`getAll`).
   - Set **Guild ID** to: `={{ $json.guildId }}`
   - Set **Channel ID** to: `={{ $json.channelId }}`
   - Set **Simplify** to `false` under options.
   - Set **OnError** to `Continue Regular Output`.

6. **Extract and Filter Attachments:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Extract New Attachments`.
   - Connect `Get Recent Messages` to `Extract New Attachments`.
   - Paste JavaScript snippet to filter messages newer than the watermark, extract attachment objects, eliminate duplicate IDs, compute file size in KB, and format message jump URLs.
   - Add a **Filter** node (`n8n-nodes-base.filter`) named `Skip Oversized Attachments`.
   - Connect `Extract New Attachments` to `Skip Oversized Attachments`.
   - Add condition: `={{ $json.fileSizeBytes }}` `<= ` `={{ $("Set Archive Settings").first().json.maxFileMb * 1024 * 1024 }}`.

7. **Configure Google Drive Folder Creation & Preparation:**
   - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`) named `Create Dated Drive Folder`.
   - Connect `Skip Oversized Attachments` to `Create Dated Drive Folder`.
   - Configure credentials (Google Drive OAuth2).
   - Set **Resource** to `Folder`.
   - Set **Folder Name** to: `={{ $("Build Channel List").first().json.runDate }}`
   - Set **Parent Folder ID** to: `={{ $("Set Archive Settings").first().json.driveParentFolderId }}`
   - Enable **Execute Once** in node parameters.
   - Add a **Code** node (`n8n-nodes-base.code`) named `Plan Uploads Into Folder`.
   - Connect `Create Dated Drive Folder` to `Plan Uploads Into Folder`.
   - Paste JavaScript snippet to sanitize filenames and attach destination directory metadata.

8. **Download and Upload Binaries:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Download Attachment`.
   - Connect `Plan Uploads Into Folder` to `Download Attachment`.
   - Set **URL** to: `={{ $json.downloadUrl }}`
   - Configure **Response Format** to `File` and set timeout to `60000` ms with batching (`5`).
   - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`) named `Upload To Drive Folder`.
   - Connect `Download Attachment` to `Upload To Drive Folder`.
   - Configure credentials (Google Drive OAuth2).
   - Set **File Name** to: `={{ $("Plan Uploads Into Folder").item.json.driveFileName }}`
   - Set **Folder ID** to: `={{ $("Create Dated Drive Folder").first().json.id }}`

9. **Record Audit Trail in Google Sheets:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Append To Archive Log`.
   - Connect `Upload To Drive Folder` to `Append To Archive Log`.
   - Configure credentials (Google Sheets OAuth2).
   - Set **Operation** to `Append` and **Mapping Mode** to `Define Below`.
   - Set **Document ID** and **Sheet Name** using expressions matching `Read Archive Log`.
   - Map columns explicitly:
     - `File Name` ➔ `={{ $("Plan Uploads Into Folder").item.json.fileName }}`
     - `Posted At` ➔ `={{ $("Plan Uploads Into Folder").item.json.postedAt }}`
     - `Posted By` ➔ `={{ $("Plan Uploads Into Folder").item.json.postedBy }}`
     - `Channel ID` ➔ `={{ $("Plan Uploads Into Folder").item.json.channelId }}`
     - `Drive Link` ➔ `={{ $json.webViewLink || "https://drive.google.com/file/d/" + $json.id + "/view" }}`
     - `Archived At` ➔ `={{ $now.toISO() }}`
     - `Content Type` ➔ `={{ $("Plan Uploads Into Folder").item.json.contentType }}`
     - `File Size KB` ➔ `={{ $("Plan Uploads Into Folder").item.json.fileSizeKb }}`
     - `Message Link` ➔ `={{ $("Plan Uploads Into Folder").item.json.messageLink }}`
     - `Drive File ID` ➔ `={{ $json.id }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Discord Bot Privileged Intent Requirement** | You must enable the **MESSAGE CONTENT** privileged intent on the Bot tab of the Discord Developer Portal. Without this intent, message attachment payloads will be omitted from API responses. |
| **Discord API Limitations** | The Discord node `getAll` operation retrieves the latest 100 messages per channel without advanced filtering. High-volume channels running past 100 messages between sweeps will experience unarchived message gaps. Increase execution frequency for active channels. |
| **Ephemeral CDN Expiration** | Discord attachment URLs are cryptographically signed and expire over time. Binaries must be downloaded within the same execution run. Do not store raw Discord CDN links for deferred processing. |