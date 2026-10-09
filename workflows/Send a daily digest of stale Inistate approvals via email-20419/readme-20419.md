Send a daily digest of stale Inistate approvals via email

https://n8nworkflows.xyz/workflows/send-a-daily-digest-of-stale-inistate-approvals-via-email-20419


# Send a daily digest of stale Inistate approvals via email

### 1. Workflow Overview

This workflow automates the process of identifying stale approval requests in Inistate and sending an email notification summarizing them. It executes daily to query records that have remained in a specific approval state longer than a defined threshold, constructs an HTML-formatted digest, and delivers it via SMTP.

The logical processing is organized into three functional blocks:
- **1.1 Schedule & Configuration:** Triggers the workflow daily at a fixed time and initializes core parameters (state name, stale threshold, recipient, and sender addresses).
- **1.2 Data Retrieval & Aggregation:** Queries the Inistate platform for pending entries older than the calculated cutoff date and aggregates the retrieved records into a single collection.
- **1.3 Content Generation & Delivery:** Transforms the aggregated entries into an HTML table and dynamic subject line, then sends the notification email using an SMTP mail server.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Configuration
- **Overview:** Initializes the workflow execution on a daily schedule and defines global parameters required for querying Inistate and addressing the notification email.
- **Nodes Involved:** 
  - `Every Morning at 9am`
  - `Set Digest Parameters`

- **Node Details:**
  - **Every Morning at 9am**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) - Time-based trigger node.
    - *Configuration Choices:* Configured to trigger daily at hour 9 (09:00).
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Output connects to `Set Digest Parameters`.
    - *Version-Specific Requirements:* v1.2.
    - *Edge Cases / Potential Failures:* Timezone discrepancies if the n8n instance timezone differs from expectations.
    - *Sub-Workflow Reference:* None.

  - **Set Digest Parameters**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Data manipulation and variable assignment node.
    - *Configuration Choices:* Assigns string and numeric values for variables (`pendingState`, `staleDays`, `digestTo`, `fromEmail`).
    - *Key Expressions or Variables:* 
      - `pendingState`: `"Pending Approval"`
      - `staleDays`: `3` (number)
      - `digestTo`: `"user@example.com"`
      - `fromEmail`: `"user@example.com"`
    - *Input/Output Connections:* Input from `Every Morning at 9am`; output connects to `Fetch Stale Inistate Entries`.
    - *Version-Specific Requirements:* v3.4.
    - *Edge Cases / Potential Failures:* Invalid email formats or incorrect state names causing subsequent queries to return empty or fail.
    - *Sub-Workflow Reference:* None.

---

#### Block 1.2: Data Retrieval & Aggregation
- **Overview:** Communicates with the Inistate API to pull stale approval records based on dynamic date calculations, then groups all individual items into a single array.
- **Nodes Involved:** 
  - `Fetch Stale Inistate Entries`
  - `Aggregate Stale Entries`

- **Node Details:**
  - **Fetch Stale Inistate Entries**
    - *Type and Technical Role:* `n8n-nodes-inistate.inistate` (v1) - External service integration node.
    - *Configuration Choices:* Retrieves all records matching specified criteria (`getAll`, `returnAll: true`). Filters results by state and calculates an update cutoff date dynamically.
    - *Key Expressions or Variables:* 
      - State filter: `={{ $('Set Digest Parameters').item.json.pendingState }}`
      - Fields returned: `state,updatedDate,updatedBy,assignees`
      - Sort direction: `asc`
      - Updated before filter: `={{ $now.minus($('Set Digest Parameters').item.json.staleDays, 'days').toISO() }}`
    - *Input/Output Connections:* Input from `Set Digest Parameters`; output connects to `Aggregate Stale Entries`.
    - *Version-Specific Requirements:* v1. Requires valid Inistate credentials and workspace/module selection.
    - *Edge Cases / Potential Failures:* API authentication failures, network timeouts, or invalid workspace/module identifiers.
    - *Sub-Workflow Reference:* None.

  - **Aggregate Stale Entries**
    - *Type and Technical Role:* `n8n-nodes-base.aggregate` (v1) - Data restructuring node.
    - *Configuration Choices:* Combines all incoming item data into a single item under the destination field `entries`.
    - *Key Expressions or Variables:* 
      - Aggregate mode: `aggregateAllItemData`
      - Destination field name: `entries`
    - *Input/Output Connections:* Input from `Fetch Stale Inistate Entries`; output connects to `Build Digest Content`.
    - *Version-Specific Requirements:* v1.
    - *Edge Cases / Potential Failures:* If zero entries are returned from Inistate, the downstream processing must handle an empty array gracefully to prevent HTML rendering errors.
    - *Sub-Workflow Reference:* None.

---

#### Block 1.3: Content Generation & Delivery
- **Overview:** Formats the collected records into an HTML table with XSS-safe text replacement, constructs a dynamic subject line, and sends the final email via SMTP.
- **Nodes Involved:** 
  - `Build Digest Content`
  - `Send Email Digest`

- **Node Details:**
  - **Build Digest Content**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Data transformation and string templating node.
    - *Configuration Choices:* Generates dynamic `subject` and `html` parameters utilizing JavaScript array mappings and sanitization helpers.
    - *Key Expressions or Variables:* 
      - Subject: `={{ $json.entries.length + ' entries waiting over ' + $('Set Digest Parameters').item.json.staleDays + ' days for approval' }}`
      - HTML body: Evaluates the `entries` array, builds an HTML table containing document IDs, relative timestamps, updating users, and assigned personnel with explicit HTML entity escaping.
    - *Input/Output Connections:* Input from `Aggregate Stale Entries`; output connects to `Send Email Digest`.
    - *Version-Specific Requirements:* v3.4.
    - *Edge Cases / Potential Failures:* Expression evaluation failures if data fields (`documentId`, `updatedDate`, etc.) are null or undefined.
    - *Sub-Workflow Reference:* None.

  - **Send Email Digest**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (v2.1) - Communication service node (SMTP).
    - *Configuration Choices:* Sends an HTML email using parameters mapped from the preceding node and configuration variables.
    - *Key Expressions or Variables:* 
      - To: `={{ $('Set Digest Parameters').item.json.digestTo }}`
      - From: `={{ $('Set Digest Parameters').item.json.fromEmail }}`
      - Subject: `={{ $json.subject }}`
      - HTML: `={{ $json.html }}`
    - *Input/Output Connections:* Input from `Build Digest Content`; terminal node in workflow.
    - *Version-Specific Requirements:* v2.1. Requires valid SMTP credentials.
    - *Edge Cases / Potential Failures:* SMTP authentication errors, rejection by the mail server due to sender address restrictions, or network timeouts.
    - *Sub-Workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every Morning at 9am` | `scheduleTrigger` | Triggers workflow daily at 9:00 AM | None | `Set Digest Parameters` | ## Daily digest of stale Inistate approvals<br><br>### How it works<br><br>This workflow runs every morning to identify stale Inistate approval entries. It sets the digest parameters, queries Inistate for entries in the pending state older than the configured threshold, aggregates the results, builds an HTML email digest, and sends it to the configured recipient.<br><br>### Setup steps<br><br>- Configure the schedule on the "Every morning" trigger to match the desired daily send time and timezone.<br>- Set the values in "Settings", including pendingState, staleDays, digestTo, and fromEmail.<br>- Connect and authorize the Inistate node so it can search the relevant approval entries.<br>- Configure email sending credentials and sender settings for the "Send digest" node.<br><br>### Customization<br><br>Adjust staleDays, pendingState, recipients, and the digest HTML/subject to match the approval process and reporting format. |
| `Set Digest Parameters` | `set` | Defines core configuration variables | `Every Morning at 9am` | `Fetch Stale Inistate Entries` | ## Schedule and settings<br><br>Starts the workflow on a daily schedule and defines the parameters used to find stale approvals and send the digest. |
| `Fetch Stale Inistate Entries` | `inistate` | Queries Inistate for stale approval records | `Set Digest Parameters` | `Aggregate Stale Entries` | ## Find and collect entries<br><br>Searches Inistate for stale pending approval entries and aggregates the returned items into a collection for the digest. |
| `Aggregate Stale Entries` | `aggregate` | Groups all retrieved entries into a single array | `Fetch Stale Inistate Entries` | `Build Digest Content` | ## Find and collect entries<br><br>Searches Inistate for stale pending approval entries and aggregates the returned items into a collection for the digest. |
| `Build Digest Content` | `set` | Constructs HTML email body and subject line | `Aggregate Stale Entries` | `Send Email Digest` | ## Build and send digest<br><br>Creates the email subject and HTML body from the collected entries, then sends the daily digest email. |
| `Send Email Digest` | `emailSend` | Sends the HTML digest via SMTP | `Build Digest Content` | None | ## Build and send digest<br><br>Creates the email subject and HTML body from the collected entries, then sends the daily digest email. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`Every Morning at 9am`).
   - Set the trigger interval rule to trigger at hour `9`.

2. **Configure Global Settings:**
   - Add a **Set** node (`Set Digest Parameters`).
   - Add string and number assignments:
     - `pendingState` (String): `"Pending Approval"`
     - `staleDays` (Number): `3`
     - `digestTo` (String): `"user@example.com"`
     - `fromEmail` (String): `"user@example.com"`
   - Connect `Every Morning at 9am` to `Set Digest Parameters`.

3. **Set Up Inistate Retrieval:**
   - Add an **Inistate** node (`Fetch Stale Inistate Entries`).
   - Select operation: `Get Many` (`getAll`), set `Return All` to true.
   - Configure parameters with expressions:
     - State: `={{ $('Set Digest Parameters').item.json.pendingState }}`
     - Fields: `state,updatedDate,updatedBy,assignees`
     - Sort Direction: `asc`
     - Updated Before: `={{ $now.minus($('Set Digest Parameters').item.json.staleDays, 'days').toISO() }}`
   - Configure your Inistate credentials, Workspace ID, and Module ID.
   - Connect `Set Digest Parameters` to `Fetch Stale Inistate Entries`.

4. **Aggregate Results:**
   - Add an **Aggregate** node (`Aggregate Stale Entries`).
   - Set aggregation mode to `Aggregate All Item Data`.
   - Set Destination Field Name to `entries`.
   - Connect `Fetch Stale Inistate Entries` to `Aggregate Stale Entries`.

5. **Build Content:**
   - Add a **Set** node (`Build Digest Content`).
   - Configure assignments:
     - `subject` (String): `={{ $json.entries.length + ' entries waiting over ' + $('Set Digest Parameters').item.json.staleDays + ' days for approval' }}`
     - `html` (String): Expression containing the HTML table generation logic with XSS character replacement.
   - Connect `Aggregate Stale Entries` to `Build Digest Content`.

6. **Configure Email Dispatch:**
   - Add a **Send Email** (SMTP) node (`Send Email Digest`).
   - Configure parameters:
     - To Email: `={{ $('Set Digest Parameters').item.json.digestTo }}`
     - From Email: `={{ $('Set Digest Parameters').item.json.fromEmail }}`
     - Subject: `={{ $json.subject }}`
     - HTML: `={{ $json.html }}`
   - Configure your SMTP credentials.
   - Connect `Build Digest Content` to `Send Email Digest`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Inistate Official Template workflow | Tag ID: `n8aEsMvLn249PJeq` |