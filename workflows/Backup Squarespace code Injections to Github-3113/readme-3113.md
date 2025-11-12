Backup Squarespace code Injections to Github

https://n8nworkflows.xyz/workflows/backup-squarespace-code-injections-to-github-3113


# Backup Squarespace code Injections to Github

### 1. Workflow Overview

This workflow automates the backup of Squarespace website header and footer code injections by fetching them from a specified Squarespace site and storing them as HTML files in a GitHub repository. It supports both manual execution and scheduled interval backups.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block:** Handles manual and scheduled triggers to start the backup process.
- **1.2 Data Retrieval Block:** Fetches the Squarespace site’s header and footer injection data via HTTP.
- **1.3 Data Cleaning Block:** Processes and cleans the raw injection HTML content for headers and footers.
- **1.4 Data Structuring Block:** Prepares the cleaned injection data with metadata for GitHub storage.
- **1.5 GitHub Backup Block:** Manages the creation or editing of injection files in the configured GitHub repository.
- **1.6 Control Flow Block:** Manages looping over multiple injection items and error handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

**Overview:**  
This block initiates the workflow either manually via a button click or automatically on a schedule (every 2 hours).

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Schedule Trigger (Scheduled Trigger)

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Allows user to manually start the workflow on demand.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Connects to "Get Squarespace data" node.  
  - Edge Cases: User must manually trigger; no automatic retries.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every 2 hours.  
  - Configuration: Interval set to 2 hours.  
  - Inputs: None  
  - Outputs: Connects to "Get Squarespace data" node.  
  - Edge Cases: Workflow will run every 2 hours; ensure API rate limits and GitHub quotas are respected.

---

#### 2.2 Data Retrieval Block

**Overview:**  
Fetches the Squarespace website data including header and footer injections by making an HTTP GET request to the configured Squarespace URL with a query parameter to request page context data.

**Nodes Involved:**  
- Get Squarespace data

**Node Details:**

- **Get Squarespace data**  
  - Type: HTTP Request  
  - Role: Retrieves the Squarespace page context data containing injections.  
  - Configuration:  
    - URL: Configured with the Squarespace site URL (default example: https://beyondspace.studio)  
    - Query Parameter: `format=page-context` to get structured page data.  
  - Inputs: From triggers ("On clicking 'execute'" or "Schedule Trigger")  
  - Outputs: Connects to "Get Header Injection" and "Get Footer Injection" nodes.  
  - Edge Cases:  
    - HTTP errors (e.g., 404 if URL is incorrect)  
    - Network timeouts  
    - Unexpected response structure  
  - Notes: User must edit the URL to their Squarespace site URL.

---

#### 2.3 Data Cleaning Block

**Overview:**  
Processes the raw HTML injections for headers and footers to remove unnecessary elements and clean the content before backup.

**Nodes Involved:**  
- Get Header Injection (Set node)  
- Clean up Headers (Code node)  
- Get Footer Injection (Set node)  
- Clean up Footers (Code node)  
- Merge Injections (Merge node)

**Node Details:**

- **Get Header Injection**  
  - Type: Set  
  - Role: Extracts header injection data from the Squarespace response and prepares metadata.  
  - Configuration:  
    - Sets `value` to `squarespace-headers` from the HTTP response JSON.  
    - Adds `id` and `name` as "headers".  
    - Adds `timestamp` as current Unix timestamp.  
    - Extracts domain from website URLs, stripping protocol and www.  
  - Inputs: From "Get Squarespace data"  
  - Outputs: Connects to "Clean up Headers"  
  - Edge Cases: Missing or malformed header data.

- **Clean up Headers**  
  - Type: Code (JavaScript)  
  - Role: Cleans the header HTML by removing Squarespace CSS links before the main content and cookie banner scripts after it.  
  - Configuration:  
    - Uses cheerio library to parse and manipulate HTML.  
    - Removes elements before a specific Squarespace CSS link and after the cookie banner script.  
    - Removes placeholder comments and excessive newlines.  
  - Inputs: From "Get Header Injection"  
  - Outputs: Connects to "Merge Injections" (second input)  
  - Edge Cases: Parsing errors, missing expected elements, fallback returns original content.

- **Get Footer Injection**  
  - Type: Set  
  - Role: Extracts footer injection data from the Squarespace response and prepares metadata.  
  - Configuration:  
    - Sets `value` to `squarespace-footers` from the HTTP response JSON.  
    - Adds `id` and `name` as "footers".  
    - Adds `timestamp` as current Unix timestamp.  
    - Extracts domain similarly as for headers.  
  - Inputs: From "Get Squarespace data"  
  - Outputs: Connects to "Clean up Footers"  
  - Edge Cases: Missing or malformed footer data.

- **Clean up Footers**  
  - Type: Code (JavaScript)  
  - Role: Cleans the footer HTML by removing social icons and elements after them.  
  - Configuration:  
    - Uses cheerio to parse HTML and remove elements after `[data-usage=social-icons-svg]`.  
    - Removes excessive newlines.  
  - Inputs: From "Get Footer Injection"  
  - Outputs: Connects to "Merge Injections" (first input)  
  - Edge Cases: Parsing errors, missing expected elements, fallback returns original content.

- **Merge Injections**  
  - Type: Merge  
  - Role: Combines cleaned header and footer injection data into a single stream for further processing.  
  - Configuration: Default merge mode (combine inputs).  
  - Inputs: From "Clean up Footers" (first input) and "Clean up Headers" (second input)  
  - Outputs: Connects to "Loop Over Items"  
  - Edge Cases: Mismatched input lengths or empty inputs.

---

#### 2.4 Data Structuring Block

**Overview:**  
Prepares global repository configuration and manages looping over each injection item for GitHub backup.

**Nodes Involved:**  
- Loop Over Items  
- Globals

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each injection item (header and footer) to process them individually.  
  - Configuration: Batch size set to 2 (processes two items per batch).  
  - Inputs: From "Merge Injections"  
  - Outputs:  
    - First output: Connects to "Globals" node  
    - Second output: Connects back to itself (loop continuation)  
  - Edge Cases: Large number of items may cause performance issues.

- **Globals**  
  - Type: Set  
  - Role: Holds GitHub repository configuration parameters for use in downstream nodes.  
  - Configuration:  
    - `repo.owner`: GitHub username (default "BeyondspaceStudio")  
    - `repo.name`: GitHub repository name (default "n8n-backup")  
    - `repo.path`: Folder path in repo, dynamically includes domain from current item (`squarespace-backup/{{ domain }}/`)  
  - Inputs: From "Loop Over Items" (second output)  
  - Outputs: Connects to "Edit Injection data"  
  - Edge Cases: User must update values to match their GitHub repo; incorrect values cause failures.

---

#### 2.5 GitHub Backup Block

**Overview:**  
Handles the creation or editing of injection HTML files in the configured GitHub repository, with error handling to create files if editing fails.

**Nodes Involved:**  
- Edit Injection data  
- If (error check)  
- Create Injection data

**Node Details:**

- **Edit Injection data**  
  - Type: GitHub  
  - Role: Attempts to edit an existing injection file in the GitHub repository.  
  - Configuration:  
    - Owner, repository, and file path dynamically set from JSON data and Globals node.  
    - File path format: `repo.path + id + ".html"` (e.g., squarespace-backup/domain/footers.html)  
    - File content: Injection HTML content.  
    - Commit message includes timestamp.  
  - Inputs: From "Globals"  
  - Outputs: Connects to "If" node  
  - Credentials: GitHub API credentials required  
  - Edge Cases:  
    - File not found (404) triggers fallback to create file  
    - API rate limits or auth errors  
    - Continue on fail enabled to allow fallback

- **If**  
  - Type: If  
  - Role: Checks if the GitHub edit operation failed due to "resource not found" error.  
  - Configuration:  
    - Condition: `$json.error` equals "The resource you are requesting could not be found"  
  - Inputs: From "Edit Injection data"  
  - Outputs:  
    - True: Connects to "Create Injection data" (file does not exist, create it)  
    - False: Connects back to "Loop Over Items" (continue processing)  
  - Edge Cases: Other errors not handled here.

- **Create Injection data**  
  - Type: GitHub  
  - Role: Creates a new injection file in the GitHub repository if editing failed.  
  - Configuration:  
    - Owner, repository, file path, content, and commit message same as "Edit Injection data".  
  - Inputs: From "If" (true branch)  
  - Outputs: Connects back to "Loop Over Items"  
  - Credentials: GitHub API credentials required  
  - Edge Cases: API errors, rate limits.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                              | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                      |
|------------------------|---------------------|----------------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger      | Manual workflow start                         | None                          | Get Squarespace data           |                                                                                                |
| Schedule Trigger        | Schedule Trigger    | Scheduled workflow start every 2 hours       | None                          | Get Squarespace data           |                                                                                                |
| Get Squarespace data    | HTTP Request        | Fetch Squarespace injections data             | On clicking 'execute', Schedule Trigger | Get Header Injection, Get Footer Injection |                                                                                                |
| Get Header Injection    | Set                 | Extract header injection data and metadata    | Get Squarespace data           | Clean up Headers               |                                                                                                |
| Clean up Headers        | Code (JavaScript)   | Clean and trim header injection HTML          | Get Header Injection           | Merge Injections              |                                                                                                |
| Get Footer Injection    | Set                 | Extract footer injection data and metadata    | Get Squarespace data           | Clean up Footers               |                                                                                                |
| Clean up Footers        | Code (JavaScript)   | Clean and trim footer injection HTML          | Get Footer Injection           | Merge Injections              |                                                                                                |
| Merge Injections        | Merge               | Combine cleaned header and footer injections  | Clean up Footers, Clean up Headers | Loop Over Items               |                                                                                                |
| Loop Over Items         | SplitInBatches      | Iterate over injection items for backup       | Merge Injections, Create Injection data | Globals, (loop continuation)  |                                                                                                |
| Globals                | Set                 | Define GitHub repo configuration parameters   | Loop Over Items                | Edit Injection data            | ## Edit this node 👇                                                                             |
| Edit Injection data     | GitHub              | Edit existing injection file in GitHub repo   | Globals                       | If                            |                                                                                                |
| If                     | If                  | Check if edit failed due to missing file      | Edit Injection data            | Create Injection data (true), Loop Over Items (false) |                                                                                                |
| Create Injection data   | GitHub              | Create new injection file in GitHub repo      | If (true branch)               | Loop Over Items               |                                                                                                |
| Sticky Note1            | Sticky Note         | Workflow purpose and setup instructions       | None                          | None                          | ## Backup to GitHub ... Each site's injections will be added into seperate folder               |
| Sticky Note2            | Sticky Note         | Main workflow loop                             | None                          | None                          | ## Main workflow loop                                                                           |
| Sticky Note4            | Sticky Note         | Instruction to edit Squarespace URL           | None                          | None                          | ## Edit this node 👇 Squarespace URL                                                            |
| Sticky Note             | Sticky Note         | Instruction to edit Globals node               | None                          | None                          | ## Edit this node 👇                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "On clicking 'execute'"  
   - No parameters  
   - Position: Start of workflow

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Schedule Trigger"  
   - Set interval to every 2 hours  
   - Position: Near manual trigger

3. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: "Get Squarespace data"  
   - Method: GET  
   - URL: Set to your Squarespace site URL (e.g., https://your-site.squarespace.com)  
   - Add query parameter: `format=page-context`  
   - Connect outputs of both triggers ("On clicking 'execute'" and "Schedule Trigger") to this node

4. **Create Set Node for Footer Injection**  
   - Type: Set  
   - Name: "Get Footer Injection"  
   - Assign fields:  
     - `value` = `{{$json["squarespace-footers"]}}`  
     - `id` = "footers"  
     - `name` = "footers"  
     - `timestamp` = `{{ new Date().getTime() }}`  
     - `domain` = `{{ ($json.website.primaryDomain || $json.website.authenticUrl || $json.website.internalUrl).replace(/^(?:https?:\/\/)?(?:www\.)?/, '') }}`  
   - Connect from "Get Squarespace data"

5. **Create Code Node to Clean Footers**  
   - Type: Code (JavaScript)  
   - Name: "Clean up Footers"  
   - Use cheerio library to parse and remove elements after `[data-usage=social-icons-svg]`  
   - Return cleaned HTML in `value` field  
   - Connect from "Get Footer Injection"

6. **Create Set Node for Header Injection**  
   - Type: Set  
   - Name: "Get Header Injection"  
   - Assign fields:  
     - `value` = `{{$json["squarespace-headers"]}}`  
     - `id` = "headers"  
     - `name` = "headers"  
     - `timestamp` = `{{ new Date().getTime() }}`  
     - `domain` = `{{ ($json.website.primaryDomain || $json.website.authenticUrl || $json.website.internalUrl).replace(/^(?:https?:\/\/)?(?:www\.)?/, '') }}`  
   - Connect from "Get Squarespace data"

7. **Create Code Node to Clean Headers**  
   - Type: Code (JavaScript)  
   - Name: "Clean up Headers"  
   - Use cheerio to remove elements before Squarespace CSS link and after cookie banner script  
   - Return cleaned HTML in `value` field  
   - Connect from "Get Header Injection"

8. **Create Merge Node**  
   - Type: Merge  
   - Name: "Merge Injections"  
   - Merge inputs from "Clean up Footers" (first input) and "Clean up Headers" (second input)  
   - Connect output to next block

9. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Name: "Loop Over Items"  
   - Batch size: 2  
   - Connect input from "Merge Injections"

10. **Create Set Node for Globals**  
    - Type: Set  
    - Name: "Globals"  
    - Assign:  
      - `repo.owner` = your GitHub username (e.g., "john-doe")  
      - `repo.name` = your GitHub repository name (e.g., "n8n-backups")  
      - `repo.path` = folder path in repo, e.g., `squarespace-backup/{{ $json.domain }}/`  
    - Connect from second output of "Loop Over Items" (loop continuation)

11. **Create GitHub Node to Edit File**  
    - Type: GitHub  
    - Name: "Edit Injection data"  
    - Operation: Edit file  
    - Owner: `={{ $json.repo.owner }}` (from Globals)  
    - Repository: `={{ $json.repo.name }}` (from Globals)  
    - File Path: `={{ $json.repo.path }}{{ $('Loop Over Items').item.json.id }}.html`  
    - File Content: `={{ $('Loop Over Items').item.json.value }}`  
    - Commit Message: `=Backup at {{ new DateTime($('Loop Over Items').item.json.timestamp) }}`  
    - Credentials: GitHub API (OAuth or Personal Access Token)  
    - Connect from "Globals" node  
    - Enable "Continue On Fail" to handle errors gracefully

12. **Create If Node to Check Edit Result**  
    - Type: If  
    - Name: "If"  
    - Condition: Check if `$json.error` equals "The resource you are requesting could not be found"  
    - Connect from "Edit Injection data"  
    - True output connects to "Create Injection data"  
    - False output connects back to "Loop Over Items" (continue processing)

13. **Create GitHub Node to Create File**  
    - Type: GitHub  
    - Name: "Create Injection data"  
    - Operation: Create file  
    - Owner, Repository, File Path, File Content, Commit Message same as "Edit Injection data"  
    - Credentials: GitHub API  
    - Connect from "If" (true output)  
    - Output connects back to "Loop Over Items"

14. **Add Sticky Notes**  
    - Add sticky notes for setup instructions and key points as per original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow backs up Squarespace header and footer injections to GitHub for version control and recovery.                                                                                                                  | Workflow purpose                                                                                 |
| Edit the HTTP Request node to set your Squarespace site URL before running.                                                                                                                                                    | Setup instruction                                                                              |
| Update the Globals node with your GitHub username, repository name, and folder path to store backups.                                                                                                                         | Setup instruction                                                                              |
| GitHub API credentials with write access to the target repository are required.                                                                                                                                               | Credential requirement                                                                         |
| The workflow supports both manual execution and scheduled backups every 2 hours.                                                                                                                                              | Usage note                                                                                     |
| For more templates by the author, visit: [My n8n Templates](https://n8n.io/creators/bangank36/)                                                                                                                               | External resource                                                                              |
| The cleaning code uses the cheerio library for robust HTML manipulation. Errors in parsing fall back to original content to avoid data loss.                                                                                | Implementation detail                                                                          |
| The workflow handles GitHub file editing failures by attempting to create the file if it does not exist, ensuring backups are always saved.                                                                                  | Error handling strategy                                                                        |

---

This documentation provides a complete understanding of the "Backup Squarespace code Injections to Github" workflow, enabling users and AI agents to reproduce, modify, and maintain it effectively.