Send WhatsApp bulk template messages from Google Sheets or CSV via WOZTELL

https://n8nworkflows.xyz/workflows/send-whatsapp-bulk-template-messages-from-google-sheets-or-csv-via-woztell-13642


# Send WhatsApp bulk template messages from Google Sheets or CSV via WOZTELL

# WhatsApp Bulk Template Messaging via WOZTELL

This n8n workflow automates the process of sending high-volume WhatsApp template messages using the **WOZTELL** platform. It supports two primary data sources: **Google Sheets** (for persistent tracking) and **CSV file uploads** (via n8n forms). The system includes built-in logic for looping, rate-limit management (delays), and status reporting to ensure reliable delivery for marketing campaigns, onboarding, or notifications.

---

### 1. Workflow Overview

The workflow is divided into two entry paths that converge into similar processing logic:

*   **1.1 Google Sheets Path (Automated/Manual):** Pulls contacts from a specific sheet where a "Sent" status is missing. It processes each row, sends the message, and updates the sheet with a "success" or "failed" status.
*   **1.2 CSV Upload Path (On-Demand):** Provides a public-facing n8n form where a user can upload a CSV. The workflow extracts the data and sends the messages in bulk.
*   **1.3 Core Logic:** Both paths utilize a batching mechanism (Split in Batches) and a "Wait" node to respect API rate limits and ensure stability.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Sheets Input & Batching
*   **Overview:** Retrieves data from a central Google Sheet and prepares it for sequential processing.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Get row(s) in sheet`, `Loop Over Items`.
*   **Node Details:**
    *   **Get row(s) in sheet:** Connects to Google Sheets via Service Account. It filters rows where the "Sent" column is empty.
    *   **Loop Over Items:** A "Split in Batches" node that ensures contacts are processed one at a time (or in small groups) to prevent overloading the WOZTELL API.

#### 2.2 WhatsApp Dispatch & Sheet Update (Google Sheets Path)
*   **Overview:** Formats the data, communicates with the WhatsApp Business API via WOZTELL, and logs the result.
*   **Nodes Involved:** `Edit Fields`, `Send message template`, `Update row in sheet`, `Wait`.
*   **Node Details:**
    *   **Edit Fields:** A "Set" node ensuring the "Phone number" is correctly mapped and available for the next step.
    *   **Send message template:** The WOZTELL node. Configured with a WABA ID and Template ID. It maps `Variable #1` to the recipient's name and `Variable #2` to the row number.
    *   **Update row in sheet:** Uses an expression `($json.ok === 1 && ...)` to determine if the message was accepted by the gateway. It writes "success" or "failed" back to the Google Sheet using the `row_number` as the key.
    *   **Wait:** Pauses for 2 seconds between messages to maintain high delivery reliability.

#### 2.3 CSV Upload & Batching (Alternative Entry)
*   **Overview:** Allows users to trigger the workflow by uploading a file through an n8n form.
*   **Nodes Involved:** `On form submission`, `Extract from File`, `Loop Over Items1`.
*   **Node Details:**
    *   **On form submission:** Displays a form titled "Upload files" requiring a `.csv` file.
    *   **Extract from File:** Parses the binary CSV data into JSON objects (Name, Phone number).
    *   **Loop Over Items1:** Controls the flow of the extracted CSV rows.

#### 2.4 WhatsApp Dispatch (CSV Path)
*   **Overview:** Sends messages to contacts derived from the CSV upload.
*   **Nodes Involved:** `Edit Fields1`, `Send message template1`, `Wait1`.
*   **Node Details:**
    *   **Send message template1:** Similar to the Google Sheets version, but usually maps variables statically or from the CSV headers.
    *   **Wait1:** A 2-second pause to prevent rate limiting.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | manualTrigger | Manual Entry | (None) | Get row(s) in sheet | 1) Starts the bulk sending process manually |
| Get row(s) in sheet | googleSheets | Data Fetching | Manual Trigger | Loop Over Items | 2) Retrieves unsent contacts from Google Sheets |
| Loop Over Items | splitInBatches | Iterator | Get row(s) in sheet, Wait | Edit Fields | 3) Loops through contacts one by one |
| Edit Fields | set | Data Mapping | Loop Over Items | Send message template | 4) Extracts the phone number for WhatsApp sending |
| Send message template | woztell | API Request | Edit Fields | Update row in sheet | 5) Sends WhatsApp template message |
| Update row in sheet | googleSheets | Status Logging | Send message template | Wait | 6) Updates delivery status in Google Sheets |
| Wait | wait | Rate Limiting | Update row in sheet | Loop Over Items | 7) Adds delay between messages |
| On form submission | formTrigger | Form Entry | (None) | Extract from File | 1) Alternative entry: Upload CSV file |
| Extract from File | extractFromFile | Data Parsing | On form submission | Loop Over Items1 | 2) Extracts contact data from uploaded CSV |
| Loop Over Items1 | splitInBatches | Iterator | Extract from File, Wait1 | Edit Fields1 | Google Sheets / n8n forms integration. |
| Edit Fields1 | set | Data Mapping | Loop Over Items1 | Send message template1 | Google Sheets / n8n forms integration. |
| Send message template1 | woztell | API Request | Edit Fields1 | Wait1 | Google Sheets / n8n forms integration. |
| Wait1 | wait | Rate Limiting | Send message template1 | Loop Over Items1 | Google Sheets / n8n forms integration. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with columns: `Phone number`, `Name`, `Sent`, and `row_number`.
    *   Obtain a WOZTELL Access Token and ensure you have an approved WhatsApp Template.
2.  **Path A: Google Sheets Setup:**
    *   Add a **Google Sheets** node (`Get row(s)`). Set the filter to `Sent` is empty.
    *   Add a **Split in Batches** node. Connect the "Loop" output to the next step.
    *   Add a **Set** node to isolate the `Phone number` and `Name`.
    *   Add the **WOZTELL** node. Select "Send Template". Map the recipient ID to the phone number and map template variables to the `Name`.
    *   Add another **Google Sheets** node (`Update`). Use the `row_number` to find the row and update the `Sent` column based on the WOZTELL response.
    *   Add a **Wait** node (2 seconds), then loop back to the **Split in Batches** node.
3.  **Path B: CSV Setup:**
    *   Add an **n8n Form Trigger**. Add a "File" field restricted to `.csv`.
    *   Add an **Extract from File** node to convert the binary upload to JSON.
    *   Follow a similar Loop/Set/WOZTELL/Wait pattern as Path A (without the "Update Sheet" step, or add a different logging method).
4.  **Credentials:**
    *   **Google Sheets:** OAuth2 or Service Account.
    *   **WOZTELL:** Header Auth (Access Token).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| WOZTELL Signup | [Platform Signup](https://platform.woztell.com/signup?lang=en&utm_campaign=plugin-n8n) |
| WhatsApp Setup Guide | [WABA Connection Guide](https://doc.woztell.com/docs/procedures/basic-whatsapp-chatbot-setup/standard-procedures-wa-connect-waba/) |
| Sample Google Sheet | [Template Sheet](https://docs.google.com/spreadsheets/d/1nw8PywwbaQSsNvsifFumWf1yMFodGrIa_EttKYM2BMM/edit?usp=sharing) |
| Access Token Guide | [WOZTELL Token Generation](https://support.woztell.com/portal/en/kb/articles/access-token#Access_Token_Generation) |
| Sample CSV File | [CSV Template](https://drive.google.com/file/d/19uI0twb8OQyeO2YoQ3x5MUntkb3v4Sve/view?usp=drive_link) |