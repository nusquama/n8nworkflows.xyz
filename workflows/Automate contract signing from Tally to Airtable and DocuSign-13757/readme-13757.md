Automate contract signing from Tally to Airtable and DocuSign

https://n8nworkflows.xyz/workflows/automate-contract-signing-from-tally-to-airtable-and-docusign-13757


# Automate contract signing from Tally to Airtable and DocuSign

This document provides a technical breakdown of the n8n workflow designed to automate the contract lifecycle from a Tally form submission through Airtable record management to DocuSign envelope distribution.

---

### 1. Workflow Overview

The workflow automates the transition from a client form submission to a legally binding signature process. It acts as a bridge between **Tally** (data collection), **Airtable** (CRM and source of truth), and **DocuSign** (e-signature).

**Logical Blocks:**
*   **1.1 Input & Normalization:** Captures the webhook, confirms receipt, and transforms the complex Tally JSON into a flat, readable object.
*   **1.2 CRM Integration & Data Enrichment:** Records the initial submission in Airtable and fetches related Service Provider details (emails, addresses) needed for the contract.
*   **1.3 Routing Logic:** Determines which contract template to use based on the user's selected "Signing Arrangement."
*   **1.4 E-Signature Execution:** Uses the DocuSign REST API to send pre-filled templates to the appropriate parties.
*   **1.5 Audit & Status Tracking:** Updates the CRM with the DocuSign Envelope ID and logs the transaction for tracking purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Normalization
Captures the initial submission and prepares the data for processing.
*   **Nodes Involved:** `Receive Tally Form Submission`, `Respond to Webhook`, `Normalize Form Payload`.
*   **Details:**
    *   **Webhook & Respond:** The webhook listens for `POST` requests. A "Respond to Webhook" node is used immediately to return a `200 OK` to Tally, preventing unnecessary retries or timeouts.
    *   **Normalization (Code Node):** Tally sends data in a nested "fields" array. This node uses JavaScript functions (`getFieldValue`, `getDropdownText`) to map specific question labels (e.g., "Subject Full Name") to standardized keys (e.g., `subject_name`).
    *   **Edge Cases:** If Tally field labels change, the Code node logic will fail to find values.

#### 2.2 CRM Integration & Data Enrichment
Stores the request and gathers external data.
*   **Nodes Involved:** `Create Airtable Contract Record`, `Lookup Service Provider in Airtable`, `Merge Contract and Provider Data`.
*   **Details:**
    *   **Create Record:** Generates a new entry in the "Contracts" table with a status of `pending`.
    *   **Lookup:** Uses the `provider_id` from the form to find the provider's email and representative details in a separate "Service Providers" table.
    *   **Merge:** Combines the new Record ID and the Provider details into a single JSON object.

#### 2.3 Preparation & Routing
Maps data to the final schema and selects the execution path.
*   **Nodes Involved:** `Map Envelope Fields`, `Route by Signing Arrangement`.
*   **Details:**
    *   **Map Envelope Fields (Set Node):** Consolidates all variables (Client, Provider, Company Reps) and captures the `contract_record_id` from the Airtable creation step.
    *   **Switch Node:** Branches the workflow into three paths: "Both Contacts", "Primary Only", or "Secondary Only".

#### 2.4 DocuSign Integration
Handles the heavy lifting of API communication with DocuSign.
*   **Nodes Involved:** `Send DocuSign Envelope (Both/Primary/Secondary)`.
*   **Details:**
    *   **HTTP Request (REST API):** Instead of the standard DocuSign node, this uses the REST API (`/v2.1/accounts/{id}/envelopes`) to allow for complex "Template Roles" and "Text Tabs". 
    *   **Pre-filling:** The workflow injects n8n variables directly into the DocuSign `textTabs`, ensuring the signer sees a pre-filled contract (Name, DOB, Address, Schedule) that they cannot edit.
    *   **Auth:** Uses OAuth2 API credentials.

#### 2.5 Post-Send Logging
Closes the loop by updating the CRM.
*   **Nodes Involved:** `Update Contract Status`, `Log Signers to Airtable`.
*   **Details:**
    *   **Update:** Changes status to `sent` and saves the `envelope_id` returned by DocuSign.
    *   **Log Signers:** Creates entries in a "Signers" table to track individual signature progress.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Tally Form Submission | Webhook | Entry Point | - | Respond to Webhook | Tally form webhook triggers the workflow. |
| Respond to Webhook | Respond to Webhook | HTTP Response | Receive Tally Form Submission | Normalize Form Payload | Immediately returns a 200 OK to prevent retries. |
| Normalize Form Payload | Code | Data Transformation | Respond to Webhook | Airtable (Create & Lookup) | Update the field labels in the Code node to match your Tally form questions. |
| Create Airtable Contract Record | Airtable | CRM Create | Normalize Form Payload | Merge Contract and Provider Data | Creates a new contract record in the Contracts table (status: pending). |
| Lookup Service Provider in Airtable | Airtable | Data Enrichment | Normalize Form Payload | Merge Contract and Provider Data | Looks up the selected service provider from the Service Providers table. |
| Merge Contract and Provider Data | Merge | Data Join | Airtable (Create & Lookup) | Map Envelope Fields | Combines both datasets for the DocuSign envelope. |
| Map Envelope Fields | Set | Data Preparation | Merge Contract and Provider Data | Route by Signing Arrangement | Maps all fields for the DocuSign API. |
| Route by Signing Arrangement | Switch | Logical Routing | Map Envelope Fields | DocuSign HTTP Nodes | Routes by signing arrangement: Both, Primary Only, or Secondary Only. |
| Send DocuSign Envelope (Both) | HTTP Request | API Integration | Route by Signing Arrangement | Update Contract Status (Both) | Sends the appropriate DocuSign envelope with pre-filled signer data. |
| Update Contract Status (Both) | Airtable | CRM Update | Send DocuSign (Both) | Log Signers to Airtable (Both) | Updates the contract record status to sent with the envelope_id. |
| Log Signers to Airtable (Both) | Airtable | Audit Logging | Update Contract Status (Both) | - | Logs signer details to the Signers table for tracking. |

*(Note: The Primary and Secondary paths follow the same structure as the "Both" path above.)*

---

### 4. Reproducing the Workflow from Scratch

1.  **Airtable Configuration:**
    *   Create a Base with three tables: `Contracts`, `Service Providers`, and `Signers`.
    *   Ensure fields match the "Airtable Setup Guide" (e.g., `provider_id` must be the primary key in the Providers table).
2.  **Tally Setup:**
    *   Create a form with fields for Subject, Primary/Secondary Contacts, and a Dropdown for "Service Provider" (whose values match Airtable `provider_id`).
    *   Set the Webhook URL to your n8n Production Webhook URL.
3.  **DocuSign Setup:**
    *   In DocuSign, create three Templates.
    *   Set Role Names exactly: `PrimaryContact`, `SecondaryContact`, `ServiceProvider`, `CompanyRep1`, `CompanyRep2`.
    *   Add "Text" fields to the templates and set their **Data Labels** to match the workflow keys (e.g., `subject_name`, `service_schedule`).
4.  **Workflow Construction:**
    *   Add a **Webhook** node (`POST`).
    *   Add a **Code** node. Copy the normalization logic and update the `getFieldValue` labels to match your Tally question text exactly.
    *   Add **Airtable** nodes. Use "Create" for the contract and "Search" for the provider lookup.
    *   Add a **Merge** node (Mode: Combine).
    *   Add a **Set** node to map all variables to clean names.
    *   Add a **Switch** node based on the string `signing_arrangement`.
    *   Add **HTTP Request** nodes for DocuSign. 
        *   URL: `https://demo.docusign.net/restapi/v2.1/accounts/YOUR_ACCOUNT_ID/envelopes`.
        *   Method: `POST`.
        *   Authentication: `OAuth2`.
        *   Body: Copy the JSON template from the original workflow, replacing `templateId` and account IDs.
    *   Add **Airtable** "Update" nodes to record the `envelopeId`.
5.  **Credentials:**
    *   Configure Airtable Personal Access Token (Scopes: `data.records:read`, `data.records:write`, `schema.bases:read`).
    *   Configure DocuSign OAuth2 (Environment: Demo or Production).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **DocuSign Environment** | Use `demo.docusign.net` for testing. Switch to `docusign.net` for production. |
| **Field Mapping** | The Code node is sensitive to Tally label changes. Always test after form edits. |
| **Airtable Setup Guide** | Detailed field list in the workflow's internal Sticky Notes. |
| **Contact & Support** | [Milo Bravo - BRaiA Labs](https://linkedin.com/in/MiloBravo/) |
| **Feedback/Consulting** | [Start the conversation here](https://tally.so/r/EkKGgB) |