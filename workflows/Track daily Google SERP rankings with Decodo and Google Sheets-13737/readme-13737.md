Track daily Google SERP rankings with Decodo and Google Sheets

https://n8nworkflows.xyz/workflows/track-daily-google-serp-rankings-with-decodo-and-google-sheets-13737


# Track daily Google SERP rankings with Decodo and Google Sheets

# Workflow Analysis: Track Daily Google SERP Rankings with Decodo and Google Sheets

This document provides a technical breakdown of an n8n workflow designed to automate the monitoring of search engine results pages (SERP). It tracks keyword rankings using the Decodo API and logs the data into Google Sheets for historical analysis.

---

### 1. Workflow Overview

The workflow automates the daily task of checking Google rankings for specific keywords. It triggers at a scheduled time, retrieves search data via Decodo (a SERP scraping provider), processes the nested JSON response into a flat structure, and filters for high-ranking results. The final data is stored in a Google Sheet, while any technical errors or empty results are logged to a separate error sheet.

**Logical Blocks:**
*   **1.1 Schedule & Configuration:** Defines the timing and search parameters (keyword, country, etc.).
*   **1.2 Data Acquisition:** Communicates with the Decodo API to fetch live Google SERP data.
*   **1.3 Result Parsing:** Unpacks the complex JSON structure to extract organic search results.
*   **1.4 Data Validation & Filtering:** Ensures only relevant, high-ranking results are kept.
*   **1.5 Storage & Error Logging:** Appends clean data to the main sheet or logs failures to an error sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
**Overview:** Sets the workflow to run daily and establishes the search criteria.
*   **Nodes Involved:** `Daily Trigger`, `Set Search Input`.
*   **Node Details:**
    *   **Daily Trigger (Schedule Trigger):** Configured to trigger every day at 09:00. 
    *   **Set Search Input (Set):** Defines variables: `keyword` (e.g., "best running shoes"), `country` ("id"), `language` ("en"), `device` ("desktop"), and `top_n` (number of top results to keep, e.g., 5).

#### 2.2 Data Acquisition
**Overview:** Uses the Decodo node to perform the actual Google search.
*   **Nodes Involved:** `Decodo Search`.
*   **Node Details:**
    *   **Decodo Search (Decodo):** Uses the `google_search` operation. It maps values from the previous "Set" node using expressions like `{{ $json.country }}`. 
    *   **Failure Handling:** "Continue on Fail" is enabled to allow the workflow to move to error-logging logic rather than crashing.

#### 2.3 Result Parsing
**Overview:** Transforms the API response into a list of individual organic results.
*   **Nodes Involved:** `Has Results`, `Split Results`, `Extract Organic`, `Split Organic`.
*   **Node Details:**
    *   **Has Results (If):** Checks if `results` is an array and contains at least one item.
    *   **Split Results (Split Out):** Breaks the initial response object into individual result items.
    *   **Extract Organic (Set):** Navigates deep JSON paths (e.g., `content.results.results.organic`) to isolate the organic listings list.
    *   **Split Organic (Split Out):** Turns the organic listings array into individual n8n items.

#### 2.4 Data Validation & Filtering
**Overview:** Formats the data and filters out results that don't meet the "Top N" or validity criteria.
*   **Nodes Involved:** `Map SERP Row`, `Valid Row`.
*   **Node Details:**
    *   **Map SERP Row (Set):** Normalizes the data. It maps `rank`, `title`, `url`, and `description`, while carrying over global data like the `keyword` and a timestamp (`$now`).
    *   **Valid Row (If):** Ensures the `url` is not empty and the `rank` is less than or equal to the `top_n` value defined in the configuration block.

#### 2.5 Storage & Error Logging
**Overview:** Saves the final data to Google Sheets.
*   **Nodes Involved:** `Write SERP Results`, `Build No-Result Error`, `Build Invalid-Row Error`, `Write SERP Errors`.
*   **Node Details:**
    *   **Write SERP Results (Google Sheets):** Appends valid rows to the `SERP_Results` sheet.
    *   **Build No-Result Error / Build Invalid-Row Error (Set):** Constructs error objects including the stage where the failure occurred, the error message, and the execution ID.
    *   **Write SERP Errors (Google Sheets):** Appends failures to the `SERP_Errors` sheet for debugging.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Trigger | Schedule Trigger | Workflow Entry | (None) | Set Search Input | Runs daily and defines the search and scrape with decodo |
| Set Search Input | Set | Config Parameters | Daily Trigger | Decodo Search | Runs daily and defines the search and scrape with decodo |
| Decodo Search | Decodo Node | SERP Scraping | Set Search Input | Has Results | Runs daily and defines the search and scrape with decodo |
| Has Results | If | Validation | Decodo Search | Split Results, Build No-Result Error | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Split Results | Split Out | Data Processing | Has Results | Extract Organic | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Extract Organic | Set | Data Extraction | Split Results | Split Organic | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Split Organic | Split Out | Data Processing | Extract Organic | Map SERP Row | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Map SERP Row | Set | Data Normalization | Split Organic | Valid Row | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Valid Row | If | Filter | Map SERP Row | Write SERP Results, Build Invalid-Row Error | Breaks the response into items, extracts organic listings, maps fields, and keeps only valid rows within the top-N ranks. |
| Write SERP Results | Google Sheets | Data Storage | Valid Row | (None) | Keeps only valid top-ranked rows and appends them to the results sheet. |
| Build No-Result Error | Set | Error Formatting | Has Results | Write SERP Errors | If there are no results or a row is missing key fields, it writes an error entry to the errors sheet. |
| Build Invalid-Row Error | Set | Error Formatting | Valid Row | Write SERP Errors | If there are no results or a row is missing key fields, it writes an error entry to the errors sheet. |
| Write SERP Errors | Google Sheets | Error Storage | Build No-Result Error, Build Invalid-Row Error | (None) | Keeps only valid top-ranked rows and appends them to the results sheet. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Schedule Trigger** node set to "Daily" at your preferred hour.
2.  **Configuration:** Add a **Set** node ("Set Search Input"). Create string variables for `keyword`, `country`, `language`, `device`, and a number variable `top_n`.
3.  **API Integration:** 
    *   Add the **Decodo** node.
    *   Configure credentials with your Decodo API Key.
    *   Set Operation to `google_search` and use expressions to pull parameters from the "Set" node.
    *   In the "Settings" tab of this node, enable **Continue on Fail**.
4.  **Initial Validation:** Add an **If** node ("Has Results") to check if `{{ $json.results.length }}` is greater than or equal to 1.
5.  **Splitting Data:**
    *   Connect the "True" output of "Has Results" to a **Split Out** node. Set "Field to Split Out" to `results`.
    *   Add a **Set** node ("Extract Organic") using a complex expression to reach the `organic` array within the Decodo payload.
    *   Add another **Split Out** node to split the `organic` field.
6.  **Mapping:** Add a **Set** node ("Map SERP Row"). Manually map fields: `keyword`, `rank` (from `pos_overall`), `url`, `title`, and `description`. Add a `checkedAt` field using `{{ $now }}`.
7.  **Filter:** Add an **If** node ("Valid Row"). 
    *   Condition 1: `url` is not empty.
    *   Condition 2: `rank` ≤ `top_n` (reference the value from the "Set Search Input" node).
8.  **Output:** 
    *   Connect the "True" output of "Valid Row" to a **Google Sheets** node. Set the operation to **Append**. Map your sheet columns to the "Map SERP Row" fields.
9.  **Error Handling:** 
    *   Connect the "False" output of "Has Results" to a **Set** node ("Build No-Result Error").
    *   Connect the "False" output of "Valid Row" to a **Set** node ("Build Invalid-Row Error").
    *   Connect both error Set nodes to a final **Google Sheets** node targeting a `SERP_Errors` sheet.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it works:** Runs on a daily schedule, calls Decodo API, extracts organic results, and validates them before saving. | Workflow Logic Overview |
| **Decodo API Documentation** | [https://decodo.io/](https://decodo.io/) (Implied by node) |
| **Setup Checklist:** Connect Google Sheets, add Decodo API credentials, edit keyword settings, and update Sheet IDs. | Setup Instructions |