Track broken backlinks with DataForSEO, log to Google Sheets, and create Asana tasks

https://n8nworkflows.xyz/workflows/track-broken-backlinks-with-dataforseo--log-to-google-sheets--and-create-asana-tasks-13689


# Track broken backlinks with DataForSEO, log to Google Sheets, and create Asana tasks

This technical documentation provides a comprehensive analysis of the n8n workflow designed to monitor broken backlinks using the DataForSEO API and automate reporting via Google Sheets and Asana.

---

### 1. Workflow Overview

The workflow automates the weekly auditing of backlink health for a specified domain. It retrieves "broken" backlinks (links pointing to 404s or other error pages) via DataForSEO, generates a dedicated report in Google Sheets, and assigns a recovery task to a team member in Asana.

**Logical Blocks:**
1.  **1.1 Data Collection Loop:** Handles scheduled triggering and paginated fetching of backlink data from DataForSEO.
2.  **1.2 Data Validation & Storage Setup:** Checks for results and initializes a new Google Spreadsheet with structured headers.
3.  **1.3 Data Transformation & Logging:** Maps the SEO metrics to spreadsheet columns and appends all detected links.
4.  **1.4 Task Notification:** Aggregates the process and creates a single summary task in Asana.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection Loop
This block manages the repetitive fetching of data to handle large volumes of backlinks (up to 5,000 items).

*   **Nodes Involved:** `Schedule Trigger`, `Initialize "items" field`, `Set "items" field`, `Get lost backlinks`, `Merge "items" with DFS response`, `Has more pages?`.
*   **Node Details:**
    *   **Schedule Trigger:** Fires every Monday at 9:00 AM.
    *   **Initialize "items" field:** Uses a Set node to create an empty array `[]`.
    *   **Get lost backlinks (DataForSEO):** Calls the Backlinks API. Configured with a `limit` of 1000 and an `offset` calculated by `{{ $runIndex * 1000 }}`. It specifically filters for `["is_broken", "=", true]`.
    *   **Has more pages? (If Node):** Controls the loop. It checks if the current `$runIndex` is less than the total count divided by 1000 and limits the maximum pages to 5 to prevent infinite loops or excessive API credit consumption.

#### 2.2 Data Validation & Storage Setup
Ensures the workflow only proceeds if broken links are found and prepares the destination file.

*   **Nodes Involved:** `Filter (has new backlinks)`, `Create spreadsheet`, `Prepare columns for Google Sheets`, `Append columns`.
*   **Node Details:**
    *   **Filter (has new backlinks):** Validates that `total_count` is greater than 0.
    *   **Create spreadsheet (Google Sheets):** Generates a new file named "Broken Backlinks for [Target Domain]".
    *   **Prepare columns for Google Sheets:** A Set node defining the schema (Date, Target, Backlink, Spam Score, etc.) with empty values to establish the header row.
    *   **Append columns:** Writes the header row to the newly created spreadsheet.

#### 2.3 Data Transformation & Logging
Processes the raw API data into a readable format for the spreadsheet.

*   **Nodes Involved:** `Set final "items" field`, `Split out items`, `Prepare data for Google Sheets`, `Append row in sheet`.
*   **Node Details:**
    *   **Set final "items" field:** Merges the previously initialized array with the full results from the loop.
    *   **Split out items:** Converts the array into individual n8n items for row-by-row processing.
    *   **Prepare data for Google Sheets:** Maps specific API fields (e.g., `url_to`, `backlink_spam_score`, `domain_from_rank`) to the spreadsheet headers.
    *   **Append row in sheet:** Appends each item as a new row in the "Broken Backlinks" tab.

#### 2.4 Task Notification
Finalizes the workflow by creating a project management entry.

*   **Nodes Involved:** `Aggregate`, `Create a task`.
*   **Node Details:**
    *   **Aggregate:** Waits for all "Append row" operations to finish and merges them back into a single execution context.
    *   **Create a task (Asana):** Creates a task in a specific Project. The task notes include a direct link to the Google Spreadsheet generated in Block 2.2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | ScheduleTrigger | Workflow Initiation | None | Initialize "items" field | |
| Initialize "items" field | Set | Data Initialization | Schedule Trigger | Set "items" field | |
| Set "items" field | Set | Loop Variable Reset | Initialize "items" field, Has more pages? | Get lost backlinks | |
| Get lost backlinks | DataForSeoBacklinksApi | SEO Data Retrieval | Set "items" field | Merge "items" with DFS response | |
| Merge "items" with DFS response | Set | Array Accumulation | Get lost backlinks | Has more pages? | |
| Has more pages? | If | Loop Control | Merge "items" with DFS response | Set "items" field, Filter (has new backlinks) | |
| Filter (has new backlinks) | Filter | Data Validation | Has more pages? | Create spreadsheet | |
| Create spreadsheet | Google Sheets | File Creation | Filter (has new backlinks) | Prepare columns for Google Sheets | |
| Prepare columns for Google Sheets | Set | Header Definition | Create spreadsheet | Append columns | |
| Append columns | Google Sheets | Header Injection | Prepare columns for Google Sheets | Set final "items" field | |
| Set final "items" field | Set | Data Flattening | Append columns | Split out items | |
| Split out items | SplitOut | Data Normalization | Set final "items" field | Prepare data for Google Sheets | |
| Prepare data for Google Sheets | Set | Field Mapping | Split out items | Append row in sheet | |
| Append row in sheet | Google Sheets | Data Insertion | Prepare data for Google Sheets | Aggregate | |
| Aggregate | Aggregate | Execution Synchronization | Append row in sheet | Create a task | |
| Create a task | Asana | Task Management | Aggregate | None | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Add a **Schedule Trigger**. Set it to "Weeks", select "Monday", and set time to "09:00".
2.  **Variable Initialization:**
    *   Add a **Set** node ("Initialize items"). Create an "Assignment" for an array named `items` with the value `={{ [] }}`.
3.  **Data Retrieval Loop:**
    *   Add a **Set** node ("Set items field") to pass the `items` array.
    *   Add a **DataForSEO** node. Select `Backlinks API` > `Get Backlinks`. Set `Target` to your domain. Set `Limit` to 1000. Use `{{ $runIndex * 1000 }}` for the `Offset`. Use a filter for `is_broken = true`.
    *   Add a **Set** node ("Merge") to combine current items: `={{ [ ...$node["Set"].json.items, ...$json.tasks[0].result[0].items] }}`.
    *   Add an **If** node ("Has more pages?"). Condition 1: `$runIndex` < `total_count / 1000`. Condition 2: `$runIndex` < 5.
    *   Connect the "True" output back to the "Set items field" node.
4.  **Spreadsheet Setup:**
    *   Connect the "False" output of the loop to a **Filter** node checking if `total_count > 0`.
    *   Add a **Google Sheets** node ("Create spreadsheet"). Configure the title and a sheet named "Broken Backlinks".
    *   Add a **Set** node ("Prepare columns") in "Raw" mode to define the JSON keys for your headers.
    *   Add a **Google Sheets** node ("Append columns") to write these headers using the Spreadsheet ID from the creation step.
5.  **Data Logging:**
    *   Add a **Set** node to finalize the array and a **Split Out** node to break the `items` array into individual objects.
    *   Add a **Set** node ("Prepare data") to map SEO fields to specific sheet columns (e.g., `$json.url_to` to "Backlink").
    *   Add a **Google Sheets** node ("Append row") using the Spreadsheet ID and the mapped data.
6.  **Task Creation:**
    *   Add an **Aggregate** node to wait for all rows to be processed.
    *   Add an **Asana** node ("Create task"). Set the task name. In the "Notes" field, use the expression `{{ $node["Create spreadsheet"].json.spreadsheetUrl }}` to provide the link.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| DataForSEO Credentials | Access your login and password at [DataForSEO API Access](https://app.dataforseo.com/api-access) |
| Documentation Context | This workflow tracks broken backlinks via the DataForSEO Backlinks API to detect link issues affecting SEO. |
| Configuration Note | Ensure Google Sheets and Asana OAuth2 credentials are pre-configured in n8n settings before deployment. |