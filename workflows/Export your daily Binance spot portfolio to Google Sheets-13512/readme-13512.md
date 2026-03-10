Export your daily Binance spot portfolio to Google Sheets

https://n8nworkflows.xyz/workflows/export-your-daily-binance-spot-portfolio-to-google-sheets-13512


# Export your daily Binance spot portfolio to Google Sheets

This document provides a technical breakdown of the n8n workflow designed to automate the export of Binance spot wallet balances to a Google Sheets spreadsheet on a daily basis.

### 1. Workflow Overview
The workflow automates the retrieval of cryptocurrency balances from a Binance account and logs them into a Google Sheet. It handles the mandatory HMAC SHA256 authentication required by Binance, filters out empty balances, and refreshes a target spreadsheet daily.

**Logical Blocks:**
*   **1.1 Trigger & Setup:** Initiates the workflow daily and defines the API credentials.
*   **1.2 Binance Authentication:** Constructs the signed query string required for the Binance API.
*   **1.3 Data Retrieval:** Fetches the account information via HTTP.
*   **1.4 Transformation & Filtering:** Extracts specific balance data and removes assets with zero holdings.
*   **1.5 Spreadsheet Update:** Clears the existing sheet and appends the new snapshot.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Setup
*   **Overview:** Sets the execution schedule and stores the necessary API keys for subsequent steps.
*   **Nodes Involved:** `Everyday at midnight`, `Set API and Signature keys`.
*   **Node Details:**
    *   **Everyday at midnight (Schedule Trigger):** Configured to trigger once per day.
    *   **Set API and Signature keys (Set):** Defines two string variables: `API_KEY` and `SIGNATURE_KEY`.
        *   *Edge Case:* If keys are incorrectly entered here, all subsequent Binance API steps will return 401/403 errors.

#### 2.2 Binance Authentication
*   **Overview:** Prepares the authenticated request. Binance requires a `timestamp` and a cryptographic `signature` for private endpoints.
*   **Nodes Involved:** `Configure timestamp query for Binance API`, `Crypto signature for Binance API`.
*   **Node Details:**
    *   **Configure timestamp (Set):** Creates a `query` string variable using the expression: `timestamp={{ Math.round(Date.now()) }}&recvWindow=50000`.
    *   **Crypto signature (Crypto):** Takes the `query` string and generates an HMAC SHA256 signature using the `SIGNATURE_KEY`.
        *   *Technical Role:* Outputs a `signature` field needed for the API URL.

#### 2.3 Data Retrieval
*   **Overview:** Executes the actual API call to Binance.
*   **Nodes Involved:** `Get Binance Account tokens`.
*   **Node Details:**
    *   **HTTP Request:** Performs a `GET` request to `https://api.binance.com/api/v3/account`.
    *   **Configuration:** Sends the API Key in the header (`X-MBX-APIKEY`) and appends the query and signature to the URL.
    *   **Error Handling:** "Retry on Fail" is enabled to handle transient network issues.

#### 2.4 Transformation & Filtering
*   **Overview:** Converts the raw JSON response into a list of specific assets and removes noise.
*   **Nodes Involved:** `Split Out tokens`, `Keep non null assets`.
*   **Node Details:**
    *   **Split Out:** Target field is `balances`. This turns a single item containing an array into multiple items (one per token).
    *   **Keep non null assets (Filter):** Uses a numerical condition to check if `free` (available balance) is greater than 0.
        *   *Requirement:* Only tokens you actually hold are passed to the spreadsheet.

#### 2.5 Spreadsheet Update
*   **Overview:** Manages the Google Sheets destination.
*   **Nodes Involved:** `Clear sheet completely`, `Append assets list in sheet`.
*   **Node Details:**
    *   **Clear sheet (Google Sheets):** Uses "Service Account" authentication to wipe the target sheet to ensure the snapshot is fresh.
    *   **Append assets (Google Sheets):** Takes the filtered list and adds it to the now-empty sheet.
        *   *Version:* 4.7.
        *   *Constraint:* Requires a valid Document ID and Sheet Name ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Everyday at midnight | Schedule Trigger | Workflow Entry | (None) | Set API and Signature keys | (None) |
| Set API and Signature keys | Set | Credential Storage | Everyday at midnight | Configure timestamp... | **Configure keys here** |
| Configure timestamp... | Set | Query Construction | Set API and Signature keys | Crypto signature... | Binance - Get portfolio assets / Set signature and make api call |
| Crypto signature... | Crypto | Authentication | Configure timestamp... | Get Binance Account tokens | Binance - Get portfolio assets / Set signature and make api call |
| Get Binance Account tokens | HTTP Request | Data Fetching | Crypto signature... | Split Out tokens | Binance - Get portfolio assets / Set signature and make api call |
| Split Out tokens | Split Out | Data Unpacking | Get Binance Account tokens | Keep non null assets | Google Sheets Storage / Keep and append only non-null values |
| Keep non null assets | Filter | Data Cleaning | Split Out tokens | Clear sheet completely | Google Sheets Storage / Keep and append only non-null values |
| Clear sheet completely | Google Sheets | Data Reset | Keep non null assets | Append assets list... | Google Sheets Storage / Keep and append only non-null values |
| Append assets list... | Google Sheets | Data Storage | Clear sheet completely | (None) | Google Sheets Storage / Keep and append only non-null values |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger:** Add a **Schedule Trigger** node set to your desired daily interval.
2.  **Config:** Add a **Set** node. Create two string variables: `API_KEY` and `SIGNATURE_KEY` (Secret Key from Binance).
3.  **Timestamp:** Add another **Set** node. Create a variable `query` with value: `timestamp={{ Math.round(Date.now()) }}&recvWindow=50000`.
4.  **Signing:** Add a **Crypto** node. 
    *   Action: `HMAC`. 
    *   Type: `SHA256`. 
    *   Value: `{{ $json.query }}`. 
    *   Secret: (Your Binance Secret Key).
    *   Property Name: `signature`.
5.  **Request:** Add an **HTTP Request** node.
    *   Method: `GET`.
    *   URL: `https://api.binance.com/api/v3/account?{{ $json.query }}&signature={{ $json.signature }}`.
    *   Headers: Add `X-MBX-APIKEY` with the value from your first Set node.
6.  **Flattening:** Add a **Split Out** node. Set the field to split to `balances`.
7.  **Filter:** Add a **Filter** node. Condition: `free` (Number) is greater than `0`.
8.  **Clear Sheet:** Add a **Google Sheets** node. Select the "Clear" operation. You must provide a Google Service Account and the specific Document/Sheet ID.
9.  **Append Data:** Add a final **Google Sheets** node. Select the "Append" operation. Map the columns from the Binance response (asset, free, locked) to your sheet headers.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **History Tracking Tip** | To keep asset history instead of snapshots, remove the "Clear sheet" node and add a "Date" column in the Append node. |
| **Binance API Documentation** | Requires "Spot" read permissions on the API key used. |
| **Target Audience** | Crypto investors, Traders, and Analysts seeking automated snapshots. |
| **Customization** | Data can be piped to Looker Studio or PowerBI once stored in Google Sheets. |