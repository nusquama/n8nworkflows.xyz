Notify new BOAMP IT tenders to a Slack channel with the BOAMP API

https://n8nworkflows.xyz/workflows/notify-new-boamp-it-tenders-to-a-slack-channel-with-the-boamp-api-13764


# Notify new BOAMP IT tenders to a Slack channel with the BOAMP API

# Reference Document: Notify new BOAMP IT tenders to Slack

## 1. Workflow Overview

This workflow automates the monitoring of French public procurement notices (BOAMP). It is designed specifically for consultants, agencies, and ESNs (Entreprises de Services du Numérique) who need to be alerted immediately when new IT-related tenders are published.

The workflow operates on a scheduled basis, queries the BOAMP Open Data API using configurable keywords, formats the results into a human-readable summary, and dispatches the notification to a designated Slack channel. It also handles "empty" states by sending a status update even if no new tenders are found, ensuring the user knows the check was performed.

### Logical Blocks
- **1.1 Input & Configuration:** Triggers the workflow and sets global variables (keywords, limits, channels).
- **1.2 Data Acquisition:** Connects to the BOAMP API to fetch the latest records.
- **1.3 Data Transformation:** Processes the raw JSON response into a formatted Slack message.
- **1.4 Notification:** Sends the final message to the Slack integration.

---

## 2. Block-by-Block Analysis

### 2.1 Input & Configuration
**Overview:** This block handles the workflow initiation (manual or scheduled) and defines the operational parameters used in subsequent steps.

*   **Nodes Involved:** `Schedule Every 6 Hours`, `Manual Trigger`, `Config`.
*   **Node Details:**
    *   **Schedule Every 6 Hours (Schedule Trigger):** Runs the workflow automatically every 6 hours.
    *   **Manual Trigger (Manual Trigger):** Allows for on-demand execution during testing or urgent checks.
    *   **Config (Edit Fields Set):**
        *   **Role:** Centralizes configuration to avoid hardcoding values in multiple nodes.
        *   **Variables:**
            *   `SLACK_CHANNEL`: Target channel (default: `#tenders`).
            *   `BOAMP_KEYWORD`: Keyword for filtering (default: `informatique`).
            *   `RESULT_LIMIT`: Number of results to fetch (default: `5`).
            *   `EMPTY_MESSAGE`: Text used when no results are found.
        *   **Edge Cases:** Incorrect channel names will cause the final Slack node to fail.

### 2.2 Data Acquisition
**Overview:** Retrieves real-time data from the official BOAMP Open Data dataset via an HTTP GET request.

*   **Nodes Involved:** `Fetch BOAMP IT Tenders`.
*   **Node Details:**
    *   **Type:** HTTP Request.
    *   **Configuration:** Performs a GET request to the `boamp-datadila.opendatasoft.com` API.
    *   **Key Expressions:**
        *   URL: Uses `{{ $json.RESULT_LIMIT }}` and `{{ $json.BOAMP_KEYWORD }}` from the Config node to build the query.
        *   Refinement: Filters by `objet` and sorts by `-dateparution` (descending date).
    *   **Failure Types:** API downtime, timeout (if result limit is set too high), or network connectivity issues.

### 2.3 Data Transformation
**Overview:** Converts the complex JSON structure from the API into a clean, formatted string suitable for Slack's Markdown-like formatting.

*   **Nodes Involved:** `Format Slack Message`.
*   **Node Details:**
    *   **Type:** Code Node (JavaScript).
    *   **Logic:**
        *   Extracts the `results` array from the previous node.
        *   If the array is empty, it returns the `EMPTY_MESSAGE`.
        *   If tenders are found, it maps through the results to extract the title (`objet`), date (`dateparution`), buyer name (`nomacheteur`), and the link (`url_avis`).
    *   **Output:** Returns an object containing the Slack channel and the constructed text block.

### 2.4 Notification
**Overview:** Delivers the processed information to the end-user.

*   **Nodes Involved:** `Send to Slack`.
*   **Node Details:**
    *   **Type:** Slack.
    *   **Operation:** Message > Post.
    *   **Configuration:** Uses the `SLACK_CHANNEL` variable for the destination and the `text` variable for the content.
    *   **Failure Types:** Invalid Slack credentials, expired OAuth token, or bot lacking permissions to post in the specified channel.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Every 6 Hours | Schedule Trigger | Workflow Initiation | None | Config | |
| Manual Trigger | Manual Trigger | Workflow Initiation | None | Config | |
| Config | Set | Parameter Definition | Schedule / Manual | Fetch BOAMP... | **Step 1 - Config** Edit channel, keyword, limit |
| Fetch BOAMP IT Tenders | HTTP Request | Data Acquisition | Config | Format Slack... | **Step 2–3 - Fetch & Format** BOAMP API → Build message (with/without results) |
| Format Slack Message | Code | Data Formatting | Fetch BOAMP... | Send to Slack | **Step 2–3 - Fetch & Format** BOAMP API → Build message (with/without results) |
| Send to Slack | Slack | Data Delivery | Format Slack... | None | **Step 4 - Send** Post to Slack |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Configuration & Triggers
1.  Create a **Manual Trigger** node.
2.  Create a **Schedule Trigger** node, set to "Every 6 Hours".
3.  Add a **Set** node (name it "Config"):
    *   Add String: `SLACK_CHANNEL` = `#tenders`.
    *   Add String: `BOAMP_KEYWORD` = `informatique`.
    *   Add Number: `RESULT_LIMIT` = `5`.
    *   Add String: `EMPTY_MESSAGE` = `Aucun appel d'offres IT trouvé pour cette exécution.`
4.  Connect both Triggers to the Config node.

### Step 2: Fetching API Data
1.  Add an **HTTP Request** node (name it "Fetch BOAMP IT Tenders").
2.  Method: `GET`.
3.  URL: `https://boamp-datadila.opendatasoft.com/api/explore/v2.1/catalog/datasets/boamp/records?limit={{ $json.RESULT_LIMIT }}&refine=objet:{{ $json.BOAMP_KEYWORD }}&sort=-dateparution`.
4.  Connect the Config node to this node.

### Step 3: Formatting Logic
1.  Add a **Code** node (name it "Format Slack Message").
2.  Language: `JavaScript`.
3.  Insert logic to check if `results` exists. If yes, loop through `results` to build a string with `objet`, `dateparution`, `nomacheteur`, and `url_avis`. If no, use the `EMPTY_MESSAGE`.
4.  Ensure the output includes the channel name and the formatted text.
5.  Connect the HTTP Request node to this node.

### Step 4: Slack Integration
1.  Add a **Slack** node.
2.  Action: `Send Message`.
3.  Select/Create your Slack Credentials (OAuth2).
4.  Channel: Use expression `{{ $json.SLACK_CHANNEL }}`.
5.  Text: Use expression `{{ $json.text }}`.
6.  Connect the Code node to this node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **BOAMP to Slack Notification:** For Consultants, ESN, and agencies who want instant Slack alerts for IT tenders. | Overview Sticky Note |
| **BOAMP Open Data API Documentation** | [Explore BOAMP Dataset](https://boamp-datadila.opendatasoft.com/explore/dataset/boamp/api/) |
| **n8n Slack Node Setup** | [n8n Slack Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/) |