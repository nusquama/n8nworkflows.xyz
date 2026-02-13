Back up and restore n8n workflows with GitHub sync

https://n8nworkflows.xyz/workflows/back-up-and-restore-n8n-workflows-with-github-sync-12721


# Back up and restore n8n workflows with GitHub sync

## 1. Workflow Overview

**Title:** Back up and restore n8n workflows with GitHub sync

**Purpose:** This workflow provides two related automations:
- **Backup (sync n8n → GitHub):** On a schedule (daily at 19:00) it exports all n8n workflows and synchronizes them into a GitHub repository as individual JSON files plus an `index.json` map. It detects **new**, **edited**, **renamed**, and **deleted** workflows and commits only necessary changes.
- **Restore (GitHub → n8n):** On manual execution it lists workflow JSON files from GitHub and recreates them in an n8n instance sequentially (useful for migration/disaster recovery).

### Logical blocks
**1.1 Backup Trigger & Repo Configuration**
- Entry via Schedule Trigger; sets GitHub repo parameters.

**1.2 Index Discovery / Initialization**
- Attempts to fetch `index.json` from GitHub; if missing, creates it and waits briefly.

**1.3 Export n8n Workflows**
- Pulls all workflows from the n8n API.

**1.4 Decide Actions (Create / Edit / Delete / Update Index)**
- Compares n8n workflows vs. `index.json` to emit action items.
- Routes items via Switch into branches.

**1.5 Create Branch (New / Renamed workflows)**
- Creates new JSON files for workflows in GitHub.

**1.6 Edit Branch (Smart edit detection)**
- Downloads existing GitHub file, parses JSON, compares normalized objects, and commits only if materially different.

**1.7 Delete Branch**
- Deletes workflow JSON files removed from n8n (or removed due to rename).

**1.8 Update Index Branch**
- Updates `index.json` when mapping changes.

**1.9 Restore Trigger & Repo Configuration**
- Entry via Manual Trigger; sets GitHub repo parameters.

**1.10 Restore Loop: List → Download → Create**
- Lists files in `workflows/`, loops one-by-one, downloads JSON, and creates workflows in n8n.
- Gracefully handles missing `workflows/` folder.

---

## 2. Block-by-Block Analysis

### 2.1 Backup Trigger & Repo Configuration
**Overview:** Starts the backup run daily and defines which GitHub repository to use for storing backups.

**Nodes Involved:**
- Schedule Trigger
- Set Github Data

**Node Details**
- **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`)
  - **Role:** Automated entry point (daily schedule).
  - **Config:** Triggers at **19:00** (hour=19).
  - **Outputs:** → `Set Github Data`
  - **Failure/edge cases:** n8n instance timezone affects “19:00”; ensure instance time is correct.
  - **Version:** 1.2

- **Set Github Data** (`n8n-nodes-base.set`)
  - **Role:** Centralizes repo owner/name for reuse by GitHub nodes.
  - **Config:** Sets:
    - `repo_owner` = `your-github-username`
    - `repo_name` = `your-github-repository-name`
  - **Key expressions:** None (static values, intended to be edited).
  - **Outputs:** → `Get Download Url for Index File`
  - **Failure/edge cases:** If not updated, GitHub calls will fail (404/not found or permission issues).
  - **Version:** 3.4

---

### 2.2 Index Discovery / Initialization
**Overview:** Loads `index.json` from GitHub to know which workflow IDs map to which file paths. If missing, creates it and waits to avoid race conditions.

**Nodes Involved:**
- Get Download Url for Index File
- Index File Not Found
- Create Index File
- Wait
- Get Index File Content

**Node Details**
- **Get Download Url for Index File** (`n8n-nodes-base.github`, operation: get file)
  - **Role:** Fetches metadata for `index.json` (not the raw file), including `download_url`.
  - **Config choices:**
    - Owner: `={{ $json.repo_owner }}`
    - Repo: `={{ $json.repo_name }}`
    - File path: `index.json`
    - Auth: GitHub OAuth2
    - **onError:** `continueErrorOutput` (important: produces an item containing `error`)
  - **Outputs:**
    - Success → `Get Index File Content`
    - Error item → `Index File Not Found` (IF checks error message)
  - **Failure/edge cases:**
    - OAuth scope/permission issues
    - Repo not found / path not found
    - API rate limiting
    - Error text matching: downstream IF expects a specific GitHub message string.
  - **Version:** 1.1

- **Index File Not Found** (`n8n-nodes-base.if`)
  - **Role:** Detects “index.json missing” based on the GitHub node error message.
  - **Condition:** `$json.error == "The resource you are requesting could not be found"`
  - **Outputs:** If true → `Create Index File`
  - **Failure/edge cases:** If GitHub changes error text or localization, condition may fail and backup will not self-initialize.
  - **Version:** 2.2

- **Create Index File** (`n8n-nodes-base.github`, operation: create file)
  - **Role:** Creates `index.json` with `{}` on first run.
  - **Config:**
    - Owner/Repo from `Set Github Data`
    - File path: `index.json`
    - Content: `{}` (empty object)
    - Commit message: `Index (Created) YYYY-MM-DD`
    - Auth: OAuth2
    - `retryOnFail: true`
  - **Outputs:** → `Wait`
  - **Failure/edge cases:** File already exists (would fail); retry won’t fix logic errors; ensure branch only runs when not found.
  - **Version:** 1.1

- **Wait** (`n8n-nodes-base.wait`)
  - **Role:** Delays for **3 seconds** to allow GitHub file creation to propagate before reading it.
  - **Config:** `amount: 3` (seconds by default in Wait node)
  - **Outputs:** → `Set Github Data` (restarts fetch flow)
  - **Failure/edge cases:** Very rarely GitHub propagation could take longer; increase wait if intermittent failures.
  - **Version:** 1.1

- **Get Index File Content** (`n8n-nodes-base.httpRequest`)
  - **Role:** Downloads raw JSON of `index.json` via `download_url`.
  - **Config:** URL `={{ $json.download_url }}`
  - **Outputs:** → `Get All Workflows`
  - **Failure/edge cases:** `download_url` may be missing if the prior GitHub call failed unexpectedly; unauthenticated raw URLs can be rate-limited in some contexts.
  - **Version:** 4.2

---

### 2.3 Export n8n Workflows
**Overview:** Retrieves the current set of workflows from the n8n instance via the n8n API.

**Nodes Involved:**
- Get All Workflows

**Node Details**
- **Get All Workflows** (`n8n-nodes-base.n8n`)
  - **Role:** Calls n8n API to list workflows for backup.
  - **Config:**
    - Filter: `excludePinnedData: true` (reduces noise / sensitive test data)
    - Credentials: **n8n API** (“Github Backup”)
    - `alwaysOutputData: true` (ensures output even in some edge conditions)
  - **Outputs:** → `C,E,D Checker`
  - **Failure/edge cases:** API key invalid; insufficient permissions; large number of workflows could increase runtime.
  - **Version:** 1

---

### 2.4 Decide Actions (Create / Edit / Delete / Update Index)
**Overview:** Compares exported workflows vs. GitHub’s `index.json` and emits a list of action items with a `status` field that drives routing.

**Nodes Involved:**
- C,E,D Checker
- Switch

**Node Details**
- **C,E,D Checker** (`n8n-nodes-base.code`)
  - **Role:** Core decision engine producing actions:
    - `create`: new workflow or renamed workflow (new name)
    - `edit`: existing workflow (later checked for real differences)
    - `delete`: removed workflow or old-name file during rename
    - `index`: update `index.json` if mapping changed
  - **Key logic/variables:**
    - Reads all workflows: `$input.all().map(i => i.json)`
    - Reads index content: `JSON.parse($('Get Index File Content').first()?.json?.data || '{}')`
    - File path convention: `workflows/${name}.json`
    - Detect rename: if `indexData[id].name !== name` → emit delete old + create new, update index
    - Detect deletions: any IDs in index but not in current workflows → delete file + remove from index
    - Only emits `index` action if serialized index differs
  - **Outputs:** → `Switch`
  - **Failure/edge cases:**
    - Workflow names containing `/` or characters invalid for GitHub paths will break file operations (e.g., create/edit/delete).
    - Duplicate workflow names cause collisions (`workflows/Same Name.json` overwritten).
    - If `Get Index File Content` is not valid JSON, `JSON.parse` will throw and stop execution.
  - **Version:** 2

- **Switch** (`n8n-nodes-base.switch`)
  - **Role:** Routes action items by `status` into 4 branches.
  - **Rules:**
    - `create` → Create branch
    - `edit` → Edit branch
    - `delete` → Delete branch
    - `index` → Update Index branch
  - **Outputs:**
    - Create → `Create New Files`
    - Edit → `Get Download Url for Github File` **and** `Merge Github & n8n File` input 2 (by position)
    - Delete → `Delete Files`
    - Update Index → `Update Index File`
  - **Failure/edge cases:** If a status is unexpected/missing, item is dropped (no default route configured).
  - **Version:** 3.3

---

### 2.5 Create Branch (New / Renamed workflows)
**Overview:** Writes new workflow JSON files into `workflows/` on GitHub with a “Created” commit message.

**Nodes Involved:**
- Create New Files

**Node Details**
- **Create New Files** (`n8n-nodes-base.github`, operation: create file)
  - **Role:** Creates `workflows/<Workflow Name>.json` for `status=create`.
  - **Config:**
    - File path: `={{ $json.path }}`
    - Content: `={{ JSON.stringify($json.data, null, 2) }}`
    - Commit message: `={{ $json.name }} (Created) YYYY-MM-DD`
    - Owner/Repo from `Set Github Data`
    - Auth: OAuth2; `retryOnFail: true`
  - **Inputs:** `Switch` (Create output)
  - **Outputs:** none further
  - **Failure/edge cases:** File already exists (create will fail); collisions from same workflow name; illegal characters in name → invalid path.
  - **Version:** 1.1

---

### 2.6 Edit Branch (Smart edit detection)
**Overview:** For `status=edit`, downloads the existing GitHub version, parses it, compares normalized JSON to the n8n-exported JSON, and commits only when meaningful differences exist.

**Nodes Involved:**
- Get Download Url for Github File
- Get Github File Content
- Parse Github File Content
- Merge Github & n8n File
- File Edit Checker
- If File Edited
- Edit Files

**Node Details**
- **Get Download Url for Github File** (`n8n-nodes-base.github`, operation: get file)
  - **Role:** Retrieves metadata (including `download_url`) for the workflow file referenced by `$json.path`.
  - **Config:** filePath `={{ $json.path }}`; owner/repo from `Set Github Data`.
  - **Inputs:** `Switch` (Edit output)
  - **Outputs:** → `Get Github File Content`
  - **Failure/edge cases:** File missing in GitHub (shouldn’t happen if index is accurate, but can after manual repo edits); rate limits.
  - **Version:** 1.1

- **Get Github File Content** (`n8n-nodes-base.httpRequest`)
  - **Role:** Downloads the raw JSON file from GitHub.
  - **Config:** URL `={{ $json.download_url }}`
  - **Outputs:** → `Parse Github File Content`
  - **Failure/edge cases:** Missing/invalid `download_url`, network errors.
  - **Version:** 4.2

- **Parse Github File Content** (`n8n-nodes-base.code`)
  - **Role:** Parses downloaded JSON string into an object `githubData`.
  - **Logic:** `githubData: JSON.parse(i.json.data || '{}')`
  - **Outputs:** → `Merge Github & n8n File` (input 1)
  - **Failure/edge cases:** Invalid JSON in repo file breaks parsing; empty content becomes `{}` and may cause false “edited” results.
  - **Version:** 2

- **Merge Github & n8n File** (`n8n-nodes-base.merge`, mode: combine by position)
  - **Role:** Combines:
    - Input 1: parsed GitHub object (`githubData`)
    - Input 2: original n8n action item from Switch Edit output (contains `data`, `path`, etc.)
  - **Config:** `combineByPosition`
  - **Outputs:** → `File Edit Checker`
  - **Failure/edge cases:** If counts/order differ between the two inputs, items may mismatch (combine-by-position is sensitive to timing/parallelism).
  - **Version:** 3.2

- **File Edit Checker** (`n8n-nodes-base.code`)
  - **Role:** Determines whether GitHub file content and n8n export differ in a meaningful way.
  - **Logic highlights:**
    - `normalize(obj)`: JSON stringifies with sorted keys for objects (not arrays).
    - Filters items where `normalize(githubData) !== normalize(data)`
    - Removes `githubData` before output so downstream keeps original structure for commit.
  - **Outputs:** → `If File Edited`
  - **Failure/edge cases:**
    - Array order is not normalized; if arrays contain the same elements in different order, will be treated as edited.
    - Some metadata fields may still change even if “functionally same”; this logic reduces but does not guarantee zero noise.
  - **Version:** 2

- **If File Edited** (`n8n-nodes-base.if`)
  - **Role:** Gate to ensure only “actually edited” items proceed.
  - **Condition used:** checks boolean `true` for expression `={{ $json.status !== null && $json.status !== undefined }}`
    - Practically, this passes if an item exists (status is present), but since `File Edit Checker` already filtered items, it acts as a redundant guard.
  - **Outputs:** true → `Edit Files`
  - **Failure/edge cases:** The condition does not explicitly validate “diff exists”; filtering is the real control.
  - **Version:** 2.2

- **Edit Files** (`n8n-nodes-base.github`, operation: edit file)
  - **Role:** Commits updated JSON to the existing GitHub file.
  - **Config:**
    - filePath `={{ $json.path }}`
    - content `={{ JSON.stringify($json.data, null, 2) }}`
    - commit message `={{ $json.name }} (Edited) YYYY-MM-DD`
  - **Inputs:** `If File Edited`
  - **Failure/edge cases:** SHA mismatch if file changed between get and edit (rare); permissions; rate limits.
  - **Version:** 1.1

---

### 2.7 Delete Branch
**Overview:** Deletes workflow JSON files from GitHub when workflows are removed from n8n or when a rename requires removing the old filename.

**Nodes Involved:**
- Delete Files

**Node Details**
- **Delete Files** (`n8n-nodes-base.github`, operation: delete file)
  - **Role:** Deletes `$json.path` from repo.
  - **Config:** commit message `={{ $json.name }} (Deleted) YYYY-MM-DD`
  - **Inputs:** `Switch` (Delete output)
  - **Failure/edge cases:** File already missing; repo protections may block deletes; permissions/rate limits.
  - **Version:** 1.1

---

### 2.8 Update Index Branch
**Overview:** If any workflow was created/renamed/deleted, updates `index.json` so future runs can detect renames and deletions correctly.

**Nodes Involved:**
- Update Index File

**Node Details**
- **Update Index File** (`n8n-nodes-base.github`, operation: edit file)
  - **Role:** Writes updated `index.json`.
  - **Config:**
    - filePath: `index.json`
    - content: `={{ JSON.stringify($json.data, null, 2) }}`
    - commit message: `Index (Edited) YYYY-MM-DD`
  - **Inputs:** `Switch` (Update Index output)
  - **Failure/edge cases:** If `index.json` was never created and error handling failed earlier, edit would fail.
  - **Version:** 1.1

---

### 2.9 Restore Trigger & Repo Configuration
**Overview:** Starts restore mode manually and defines which GitHub repository contains the backup files.

**Nodes Involved:**
- When clicking ‘Execute workflow’
- Set Github Data1

**Node Details**
- **When clicking ‘Execute workflow’** (`n8n-nodes-base.manualTrigger`)
  - **Role:** Manual entry point for restore.
  - **Outputs:** → `Set Github Data1`
  - **Version:** 1

- **Set Github Data1** (`n8n-nodes-base.set`)
  - **Role:** Same as backup: repo owner/name for restore path.
  - **Outputs:** → `List Workflow Files`
  - **Version:** 3.4

---

### 2.10 Restore Loop: List → Download → Create
**Overview:** Lists `workflows/` folder in GitHub, loops through each file sequentially, downloads JSON, and creates workflows in n8n.

**Nodes Involved:**
- List Workflow Files
- Workflows Folder Not Found
- Loop Over Items
- Get File Content
- Create Workflow

**Node Details**
- **List Workflow Files** (`n8n-nodes-base.github`, operation: list)
  - **Role:** Lists files under `workflows/`.
  - **Config:**
    - filePath: `workflows/`
    - Auth: OAuth2
    - **onError:** continueErrorOutput (sends error item onward)
  - **Outputs:**
    - Main list output → `Loop Over Items`
    - Error output → `Workflows Folder Not Found`
  - **Failure/edge cases:** Folder absent (common on first run before any backups); large folder listing.
  - **Version:** 1.1

- **Workflows Folder Not Found** (`n8n-nodes-base.if`)
  - **Role:** Detects missing folder by matching error message.
  - **Condition:** `$json.error == "The resource you are requesting could not be found"`
  - **Outputs:** (no further nodes connected) effectively stops restore gracefully.
  - **Failure/edge cases:** Same error-text fragility as index check.
  - **Version:** 2.2

- **Loop Over Items** (`n8n-nodes-base.splitInBatches`)
  - **Role:** Processes files sequentially to reduce conflicts and rate-limit risk.
  - **Config:** default batch settings (not explicitly set).
  - **Connections:**
    - Output 1 (next batch) is connected to nothing
    - Output 2 (current batch item) → `Get File Content`
    - After `Create Workflow`, it loops back to `Loop Over Items` to continue.
  - **Failure/edge cases:** If no “continue” output is used properly, looping patterns can behave unexpectedly; here loopback from `Create Workflow` keeps iteration going.
  - **Version:** 3

- **Get File Content** (`n8n-nodes-base.httpRequest`)
  - **Role:** Downloads each workflow JSON file using `download_url` from list results.
  - **Config:** URL `={{ $json.download_url }}`
  - **Outputs:** → `Create Workflow`
  - **Failure/edge cases:** If list result doesn’t include `download_url` (rare), request fails.
  - **Version:** 4.2

- **Create Workflow** (`n8n-nodes-base.n8n`, operation: create)
  - **Role:** Creates a new workflow in n8n from downloaded JSON.
  - **Config:** `workflowObject: ={{ $json.data }}`
  - **Outputs:** → `Loop Over Items` (continue loop)
  - **Failure/edge cases:**
    - Creating duplicates if workflows already exist (no de-duplication by name/ID).
    - Some workflow JSON may reference credentials that don’t exist in the target instance; creation may fail or nodes may be invalid.
    - Version differences between source and target n8n/node versions can cause import issues.
  - **Version:** 1

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Backup entry point (daily 19:00) | — | Set Github Data | # n8n-Workflow-Github-Backup |
| Set Github Data | set | Define backup repo owner/name | Schedule Trigger; Wait | Get Download Url for Index File | # n8n-Workflow-Github-Backup |
| Get Download Url for Index File | github (file:get) | Fetch index.json metadata / download_url | Set Github Data | Get Index File Content; Index File Not Found | # n8n-Workflow-Github-Backup |
| Index File Not Found | if | Detect missing index.json | Get Download Url for Index File | Create Index File | # n8n-Workflow-Github-Backup |
| Create Index File | github (file:create) | Create index.json `{}` | Index File Not Found | Wait | # n8n-Workflow-Github-Backup |
| Wait | wait | Delay after creating index.json | Create Index File | Set Github Data | # n8n-Workflow-Github-Backup |
| Get Index File Content | httpRequest | Download raw index.json | Get Download Url for Index File | Get All Workflows | # n8n-Workflow-Github-Backup |
| Get All Workflows | n8n (list) | Export workflows from n8n API | Get Index File Content | C,E,D Checker | # n8n-Workflow-Github-Backup |
| C,E,D Checker | code | Decide create/edit/delete/index actions | Get All Workflows | Switch | # n8n-Workflow-Github-Backup |
| Switch | switch | Route actions by status | C,E,D Checker | Create New Files; Get Download Url for Github File; Merge Github & n8n File; Delete Files; Update Index File | # n8n-Workflow-Github-Backup |
| Create New Files | github (file:create) | Commit new workflow JSON files | Switch (Create) | — | # n8n-Workflow-Github-Backup |
| Get Download Url for Github File | github (file:get) | Fetch existing workflow file download_url | Switch (Edit) | Get Github File Content | # n8n-Workflow-Github-Backup |
| Get Github File Content | httpRequest | Download existing workflow JSON from GitHub | Get Download Url for Github File | Parse Github File Content | # n8n-Workflow-Github-Backup |
| Parse Github File Content | code | Parse GitHub JSON into object | Get Github File Content | Merge Github & n8n File | # n8n-Workflow-Github-Backup |
| Merge Github & n8n File | merge | Combine GitHub + n8n versions by position | Parse Github File Content; Switch (Edit) | File Edit Checker | # n8n-Workflow-Github-Backup |
| File Edit Checker | code | Normalize & compare; filter unchanged | Merge Github & n8n File | If File Edited | # n8n-Workflow-Github-Backup |
| If File Edited | if | Gate edits (redundant after filter) | File Edit Checker | Edit Files | # n8n-Workflow-Github-Backup |
| Edit Files | github (file:edit) | Commit edited workflow JSON | If File Edited | — | # n8n-Workflow-Github-Backup |
| Delete Files | github (file:delete) | Delete removed/old-name workflow files | Switch (Delete) | — | # n8n-Workflow-Github-Backup |
| Update Index File | github (file:edit) | Commit updated index.json | Switch (Update Index) | — | # n8n-Workflow-Github-Backup |
| When clicking ‘Execute workflow’ | manualTrigger | Restore entry point | — | Set Github Data1 | # n8n-Workflow-Github-Restore |
| Set Github Data1 | set | Define restore repo owner/name | When clicking ‘Execute workflow’ | List Workflow Files | # n8n-Workflow-Github-Restore |
| List Workflow Files | github (file:list) | List `workflows/` folder | Set Github Data1 | Loop Over Items; Workflows Folder Not Found | # n8n-Workflow-Github-Restore |
| Workflows Folder Not Found | if | Detect missing workflows folder | List Workflow Files (error) | — | # n8n-Workflow-Github-Restore |
| Loop Over Items | splitInBatches | Sequentially process workflow files | List Workflow Files; Create Workflow | Get File Content | # n8n-Workflow-Github-Restore |
| Get File Content | httpRequest | Download each workflow JSON file | Loop Over Items | Create Workflow | # n8n-Workflow-Github-Restore |
| Create Workflow | n8n (create) | Create workflow in n8n from JSON | Get File Content | Loop Over Items | # n8n-Workflow-Github-Restore |

---

## 4. Reproducing the Workflow from Scratch

### A) Credentials to create first
1. **GitHub OAuth2 credential**
   - In n8n: *Credentials → New → GitHub OAuth2 API*
   - In GitHub: *Settings → Developer settings → OAuth Apps → New OAuth App*
   - Set **Authorization callback URL** to your n8n base URL (as required by n8n’s credential UI).
   - Ensure the token has permissions to read/write repository contents (private repos require appropriate access).

2. **n8n API credential**
   - In n8n: *Settings → API* → create an API key.
   - In n8n: *Credentials → New → n8n API* (or “n8nApi” credential type depending on version) and store:
     - Base URL of your n8n instance
     - API key

### B) Backup (n8n → GitHub) flow
1. Create **Schedule Trigger**
   - Set to run daily at **19:00**.

2. Create **Set** node named **Set Github Data**
   - Add string fields:
     - `repo_owner` = your GitHub username/org
     - `repo_name` = your repo name

3. Create **GitHub** node named **Get Download Url for Index File**
   - Resource: **File**
   - Operation: **Get**
   - Owner: expression `{{$json.repo_owner}}`
   - Repository: expression `{{$json.repo_name}}`
   - File path: `index.json`
   - Enable **Continue On Fail** (or set node “On Error” to continue)
   - Attach **GitHub OAuth2** credential.

4. Create **HTTP Request** node named **Get Index File Content**
   - URL: expression `{{$json.download_url}}`
   - Response: default (must include `.json.data` as the body string)

5. Create **IF** node named **Index File Not Found**
   - Condition: String equals:
     - Left: `{{$json.error}}`
     - Right: `The resource you are requesting could not be found`

6. Create **GitHub** node named **Create Index File**
   - Resource: **File**
   - Operation: **Create**
   - File path: `index.json`
   - File content: `{}`
   - Commit message: `Index (Created) {{ new Date().toISOString().split('T')[0] }}`
   - Attach GitHub OAuth2.

7. Create **Wait** node named **Wait**
   - Wait: **3 seconds**

8. Create **n8n** node named **Get All Workflows**
   - Operation: **Get Many / List workflows** (wording depends on n8n version)
   - Set filter **Exclude pinned data** = true
   - Attach **n8n API** credential.

9. Create **Code** node named **C,E,D Checker**
   - Paste the logic that:
     - Parses index from `Get Index File Content`
     - Compares with `$input.all()` workflows
     - Emits items with `status`, `path`, `name`, and `data` (workflow JSON)
     - Optionally emits an `index` item with updated map

10. Create **Switch** node named **Switch**
   - Route on expression `{{$json.status}}`
   - Create 4 outputs named:
     - Create: equals `create`
     - Edit: equals `edit`
     - Delete: equals `delete`
     - Update Index: equals `index`

11. Create **GitHub** node named **Create New Files**
   - Operation: **Create**
   - File path: `{{$json.path}}`
   - Content: `{{ JSON.stringify($json.data, null, 2) }}`
   - Commit message: `{{ $json.name }} (Created) {{ new Date().toISOString().split('T')[0] }}`

12. Create **Edit branch nodes**
   - **GitHub**: **Get Download Url for Github File** (operation Get, filePath `{{$json.path}}`)
   - **HTTP Request**: **Get Github File Content** (url `{{$json.download_url}}`)
   - **Code**: **Parse Github File Content** (JSON.parse to `githubData`)
   - **Merge**: **Merge Github & n8n File**
     - Mode: Combine
     - Combine by: Position
   - **Code**: **File Edit Checker**
     - Normalize/sort keys and filter equal objects out
   - **IF**: **If File Edited** (can be simplified; keep consistent with original if desired)
   - **GitHub**: **Edit Files** (operation Edit, filePath `{{$json.path}}`, content stringified, commit message “(Edited) …”)

13. Create **GitHub** node named **Delete Files**
   - Operation: **Delete**
   - File path: `{{$json.path}}`
   - Commit message: `{{ $json.name }} (Deleted) {{ new Date().toISOString().split('T')[0] }}`

14. Create **GitHub** node named **Update Index File**
   - Operation: **Edit**
   - File path: `index.json`
   - Content: `{{ JSON.stringify($json.data, null, 2) }}`
   - Commit message: `Index (Edited) {{ new Date().toISOString().split('T')[0] }}`

15. Wire connections (backup)
   1. Schedule Trigger → Set Github Data
   2. Set Github Data → Get Download Url for Index File
   3. Get Download Url for Index File:
      - Success → Get Index File Content → Get All Workflows → C,E,D Checker → Switch
      - Error → Index File Not Found → Create Index File → Wait → Set Github Data
   4. Switch outputs:
      - Create → Create New Files
      - Edit → Get Download Url for Github File
      - Edit (same items) → Merge Github & n8n File input 2 (n8n side)
      - Delete → Delete Files
      - Update Index → Update Index File
   5. Get Download Url for Github File → Get Github File Content → Parse Github File Content → Merge Github & n8n File input 1 → File Edit Checker → If File Edited → Edit Files

### C) Restore (GitHub → n8n) flow
1. Create **Manual Trigger** named **When clicking ‘Execute workflow’**
2. Create **Set** named **Set Github Data1** with `repo_owner`, `repo_name`
3. Create **GitHub** node **List Workflow Files**
   - Resource: File
   - Operation: List
   - File path: `workflows/`
   - Enable continue on fail
4. Create **IF** node **Workflows Folder Not Found**
   - Same “error equals resource not found” check
   - Leave unconnected or connect to a notification/log if desired
5. Create **Split In Batches** node **Loop Over Items**
   - Default settings are fine (sequential loop pattern)
6. Create **HTTP Request** node **Get File Content**
   - URL: `{{$json.download_url}}`
7. Create **n8n** node **Create Workflow**
   - Operation: Create workflow
   - Workflow object: `{{$json.data}}`
8. Wire connections (restore)
   - Manual Trigger → Set Github Data1 → List Workflow Files
   - List Workflow Files (success) → Loop Over Items
   - List Workflow Files (error) → Workflows Folder Not Found
   - Loop Over Items (current batch output) → Get File Content → Create Workflow → back to Loop Over Items

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “n8n-Workflow-Github-Backup” | Sticky note title for the backup section |
| “n8n-Workflow-Github-Restore” | Sticky note title for the restore section |
| GitHub OAuth setup steps and requirement to connect credentials to all GitHub nodes | From “Setup Requirements” sticky note |
| `index.json` structure maps workflow IDs to `{ name, file_path }` and enables rename/delete detection | From “Index System” sticky note |
| Smart edit detection normalizes JSON to reduce commit noise due to n8n metadata changes | From “Smart Edit Detection” sticky note |
| Commit message convention: `[Workflow Name] ([Action]) YYYY-MM-DD` and examples | From “Commit Messages” sticky note |
| Restore runs sequentially and stops gracefully if `workflows/` folder doesn’t exist; run backup at least once first | From “How It Works” sticky note |